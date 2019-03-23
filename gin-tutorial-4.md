
## Overview

在上一篇教程[《[Gin教程-3]引入Redis》](https://marcoma.xyz/2019/02/28/gin-tutorial-3/)里，我们引入了`Redis`，实现了`Redis utils`且结合腾讯云`COS`开发了图片上传的功能。

在本篇教程中，我们要开始开发API层，接收客户端的请求，解析请求里的参数，然后对各个Model执行增/删/改/查操作。

## 层次结构

API层包含了很多API，每个API的职责就是：

1. 解析用户请求的参数（表单、url参数......）
2. 验证参数有效性
3. 调用对应Model的增/删/改/查方法，得到执行结果
4. 对执行结果进行封装，返回给用户

但是在真正调用API前，很多时候我们需要做一些额外的工作。拿新增图片这个操作为例，我们不能允许用户在未登录状态下新增图片，所以调用`AddPhoto()`前我们还得额外地“鉴权”，去查查该用户是不是已经登录了，所以我们需要写一个“鉴权的方法”；再例如查询一个bucket下的所有图片时，我们需要对数据分页，总不能一次请求返回几万张图片数据吧？所以我们需要写一个“分页的方法”。

所以，实际上API层是由中间件 + API组成的，**每一个中间件其实可以看成一个特殊的API，它也要解析参数，也要执行相应的逻辑，也要返回结果给下一级，称其为中间件是因为它不是执行链的最后一个，而是在执行链的中间位置。**

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-mdw.png)

如上图所示，在请求到达最终执行业务逻辑的API前，很可能要先经过一系列的中间件，就像是一条执行链。

## 分页中间件
首先来看看分页中间件，凡是返回多项数据的API，都必须先经过分页。例如`GetPhotoByBucketID()`、`GetBucketByAuthID()`，我们要把请求中的`page`参数先转化为`offset = page * PAGE_SIZE`，然后才能从数据库中直接利用offset得到我们想要的数据。

在项目根目录下新建`middleware`文件夹，然后新建`pagination.go`代码文件。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
	|—— ......
|—— middleware
	|—— pagination.go
```
接着可以实现分页中间件：

```golang
package middleware

import (
	"gin-photo-storage/constant"
	"github.com/gin-gonic/gin"
	"github.com/pkg/errors"
	"net/http"
	"strconv"
)

var InvalidPageNoError = errors.New("page no can not be negative")

// A wrapper function which returns the pagination middleware.
func GetPaginationMiddleware() func(*gin.Context) {
	return func(context *gin.Context) {
		responseCode := constant.PAGINATION_SUCCESS
		pageNo := context.Query("page")
		if pageNo == "" {
			responseCode = constant.INVALID_PARAMS
		} else {
			pageOffset, err := GetPagination(pageNo)
			if err != nil {
				responseCode = constant.INVALID_PARAMS
			} else {
				context.Set("offset", pageOffset)
			}
		}

		if responseCode == constant.INVALID_PARAMS {
			data := make(map[string]string)
			data["page"] = pageNo
			context.JSON(http.StatusBadRequest, gin.H{
				"code": constant.INVALID_PARAMS,
				"data": data,
				"msg":  constant.GetMessage(constant.INVALID_PARAMS),
			})
			context.Abort()
		}

		context.Next()
	}
}

// Pagination function which calculates the offset given the page number.
func GetPagination(pageNo string) (int, error) {
	pageNoInt, err := strconv.Atoi(pageNo)
	if err != nil {
		return 0, err
	}
	if pageNoInt < 0 {
		return 0, InvalidPageNoError
	}
	return pageNoInt * constant.PAGE_SIZE, nil
}
```
**Note：为什么我们要绕一圈，定义一个wrapper方法？**用wrapper方法返回一个`func(context *gin.Context)`，而不是直接把分页中间件定义成一个`func(context *gin.Context)`？因为用wrapper方法的话，我们可以在return真正的中间件方法前做一些其他操作，例如：

```golang
func GetXXXMiddleware() func(*gin.Context) {
	// ......
	// 可以在这连接数据库
	// 可以在这读某个磁盘文件
	// ......
	return func(context *gin.Context) {
		// ......
		// 真正的中间件逻辑
		// ......
	}
}
```
把一些耗时的操作，或者那些只需要在最开始做一次的操作放在return中间件的前面，就不需要把它们写在中间件的内部，“污染”中间件真正的逻辑。

## Auth中间件
至于Auth中间件，我们要实现的逻辑是：

1. 尝试从请求的cookie里获取`JWT`字符串，如果不存在`JWT`串说明该用户从未登录，或者N久前登录导致cookie都消失了。
2. 调用`utils`里的解析方法，解析该`JWT`串，得到`UserClaim`。
3. 从`UserClaim`里获取用户名，检查`Redis`里存不存在该用户名，如果不存在说明该用户登录状态已过期。
4. 若前几步都验证成功，可以把用户名加到本次请求的`context`里，完成本中间件的所有任务，去做下一步。

我们在`gin-photo-storage/middleware/`下新建`auth.go`。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
	|—— ......
|—— middleware
	|—— pagination.go
	|—— auth.go
```
然后可以实现`auth.go`：

```golang
package middleware

import (
	"gin-photo-storage/constant"
	"gin-photo-storage/utils"
	"github.com/gin-gonic/gin"
	"log"
	"net/http"
)

// A wrapper function which returns the auth middleware.
func GetAuthMiddleware() gin.HandlerFunc {
	return func(context *gin.Context) {
		jwtString, err := context.Cookie(constant.JWT)
		if err != nil {
			log.Println(err)
			context.JSON(http.StatusBadRequest, gin.H{
				"code": constant.JWT_MISSING_ERROR,
				"data": make(map[string]string),
				"msg": constant.GetMessage(constant.JWT_MISSING_ERROR),
			})
			context.Abort()
			return
		}

		claim, err := utils.ParseJWT(jwtString)
		if err != nil {
			log.Println(err)
			context.JSON(http.StatusBadRequest, gin.H{
				"code": constant.JWT_PARSE_ERROR,
				"data": make(map[string]string),
				"msg": constant.GetMessage(constant.JWT_PARSE_ERROR),
			})
			context.Abort()
			return
		}

		if utils.IsAuthInRedis(claim.UserName) {
			context.Set("user_name", claim.UserName)
			context.Next()
		} else {
			context.JSON(http.StatusBadRequest, gin.H{
				"code": constant.USER_AUTH_TIMEOUT,
				"data": make(map[string]string),
				"msg": constant.GetMessage(constant.USER_AUTH_TIMEOUT),
			})
			context.Abort()
		}
	}
}
```
**Note：**

+ **context就是本次请求的上下文数据，它的生命周期就是整个请求的生命周期。只要请求没完成/没结束，context都会存在。所以我们才能把用户名set到context里，传给下一步。**
+ **注意在验证异常时，除了调用`context.Abort()`来终止执行链之外，还要及时return，避免执行该中间件下面的逻辑。**

## Refresh中间件
**所谓的Refresh中间件，所要refresh的就是用户的登录token。**试想一下这样的场景：用户在12:00登录了系统，根据我们设定的`JWT_EXP_MINUTE = 30`，说明他的登录状态在12:31就在`Redis`里过期了，必须得重新登录。虽然过期时间设短一点相对会安全些，但是这样是非常影响用户体验的。

所以，我们就要设计refresh机制。**如果用户在登录之后有各种频繁的操作，那我们就不断地给他刷新他的token有效期。**例如用户在12:00登录了，token有效期到12:30，但是他在12:18新建了bucket，那我们就把他的token有效期刷新，刷成12:48，之后他又在12:35添加了20张图片，那就再把他的token有效期刷到13:05，依此类推。

先在`gin-photo-gallery/middleware/`下新建`refresh.go`代码文件。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
	|—— ......
|—— middleware
	|—— pagination.go
	|—— auth.go
	|—— refresh.go
```
然后可以实现`refresh.go`：

```golang
package middleware

import (
	"gin-photo-storage/conf"
	"gin-photo-storage/constant"
	"gin-photo-storage/utils"
	"github.com/gin-gonic/gin"
	"log"
	"net/http"
)

// A wrapper function which returns the refresh middleware.
func GetRefreshMiddleware() gin.HandlerFunc {
	return func(context *gin.Context) {
		if userName, ok := context.Get("user_name"); ok {
			// generate a new valid JWT for the user
			jwtString, err := utils.GenerateJWT(userName.(string))
			if err != nil {
				log.Fatalln(err)
				data := make(map[string]string)
				data["user_name"] = userName.(string)
				context.JSON(http.StatusBadRequest, gin.H{
					"code": constant.JWT_GENERATION_ERROR,
					"data": data,
					"msg": constant.GetMessage(constant.JWT_GENERATION_ERROR),
				})
				context.Abort()
				return
			}

			// save the new JWT in user's cookie
			context.SetCookie(constant.JWT, jwtString,
				constant.COOKIE_MAX_AGE, "/",
				conf.ServerCfg.Get(constant.SERVER_DOMAIN), true, true)

			// refresh user in the redis
			err = utils.AddAuthToRedis(userName.(string))
			if err != nil {
				log.Fatalln(err)
				context.JSON(http.StatusBadRequest, gin.H{
					"code": constant.INTERNAL_SERVER_ERROR,
					"data": make(map[string]string),
					"msg": constant.GetMessage(constant.INTERNAL_SERVER_ERROR),
				})
				context.Abort()
				return
			}
			context.Next()
		}
		context.Abort()
	}
}
```
**Note：我们不仅要为用户颁发一个新的`JWT`，还要在`Redis`里给用户刷新一下，避免过期。**

## Auth API
把几个必要的中间件实现完，可以开始实现我们的业务逻辑了，首先来看看Auth API。

先在项目根目录下新建`apis`文件夹，再在`apis`下新建`v1`文件夹，代表现在开发的API版本是v1。最后在`apis/v1/`下新建`auth.go`代码文件。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
	|—— ......
|—— middleware
	|—— ......
|—— apis
    |—— v1
        |—— auth.go
```
因为在API的逻辑里要对请求里的参数作合法性验证，所以我们就采用了第三方的`beego validation`来实现参数验证。（不使用第三方库，纯手写验证逻辑也不是不可以，只不过用库会方便优雅一点）

+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u github.com/astaxie/beego/validation`
+ 否则可用go mod来安装 

在`auth.go`里，我们要实现2个API：用户注册（AddAuth）和用户登录验证（CheckAuth）：

```golang
package v1

import (
	"gin-photo-storage/conf"
	"gin-photo-storage/models"
	"gin-photo-storage/constant"
	"gin-photo-storage/utils"
	"github.com/astaxie/beego/validation"
	"github.com/gin-gonic/gin"
	"log"
	"net/http"
)

// Add a new auth.
func AddAuth(context *gin.Context) {

	userName := context.PostForm("user_name")
	password := context.PostForm("password")
	email := context.PostForm("email")

	// set up param validation
	validCheck := validation.Validation{}
	validCheck.Required(userName, "user_name").Message("Must have user name")
	validCheck.MaxSize(userName, 16, "user_name").Message("User name length can not exceed 16")
	validCheck.MinSize(userName, 6, "user_name").Message("User name length is at least 6")
	validCheck.Required(password, "password").Message("Must have password")
	validCheck.MaxSize(password, 16, "password").Message("Password length can not exceed 16")
	validCheck.MinSize(password, 6, "password").Message("Password length is at least 6")
	validCheck.Required(email, "email").Message("Must have email")
	validCheck.MaxSize(email, 128, "email").Message("Email can not exceed 128 chars")

	responseCode := constant.INVALID_PARAMS
	if !validCheck.HasErrors() {
		if err := models.AddAuth(userName, password, email); err == nil {
			responseCode = constant.USER_ADD_SUCCESS
		} else {
			responseCode = constant.USER_ALREADY_EXIST
		}
	} else {
		for _, err := range validCheck.Errors {
			log.Println(err)
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": userName,
		"msg": constant.GetMessage(responseCode),
	})
}

// Check if an auth is valid.
func CheckAuth(context *gin.Context) {

	userName := context.PostForm("user_name")
	password := context.PostForm("password")

	// set up param validation
	validCheck := validation.Validation{}
	validCheck.Required(userName, "user_name").Message("Must have user name")
	validCheck.MaxSize(userName, 16, "user_name").Message("User name length can not exceed 16")
	validCheck.MinSize(userName, 6, "user_name").Message("User name length is at least 6")
	validCheck.Required(password, "password").Message("Must have password")
	validCheck.MaxSize(password, 16, "password").Message("Password length can not exceed 16")
	validCheck.MinSize(password, 6, "password").Message("Password length is at least 6")

	responseCode := constant.INVALID_PARAMS
	if !validCheck.HasErrors() {
		if models.CheckAuth(userName, password) {
			if jwtString, err := utils.GenerateJWT(userName); err != nil {
				responseCode = constant.JWT_GENERATION_ERROR
			} else {
				// pass auth validation
				// 1. set JWT to user's cookie
				// 2. add user to the Redis
				context.SetCookie(constant.JWT, jwtString,
					constant.COOKIE_MAX_AGE, conf.ServerCfg.Get(constant.SERVER_PATH),
					conf.ServerCfg.Get(constant.SERVER_DOMAIN), true, true)
				if err = utils.AddAuthToRedis(userName); err != nil {
					responseCode = constant.INTERNAL_SERVER_ERROR
				} else {
					responseCode = constant.USER_AUTH_SUCCESS
				}
			}
		} else {
			responseCode = constant.USER_AUTH_ERROR
		}
	} else {
		for _, err := range validCheck.Errors {
			log.Println(err)
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": userName,
		"msg": constant.GetMessage(responseCode),
	})
}
```
Auth API里的核心点在于业务逻辑，在`CheckAuth()`里如果一个用户验证通过，不仅要为他“颁发”一个`JWT`串，还要把该用户写入到`Redis`里，用`Redis`来记录这个用户是否在有效登录期内。如果一个用户请求的cookie里带有`JWT`，但是`Redis`里没有该用户的数据，要么是他伪造了`JWT`，要么是`JWT`和`Redis`里的数据过期时间不同步。

**Note：**

+ **代码里用的是`context.PostForm(xxx)`来获取请求参数，这个前提是用户会用POST表单来传参，如果用户用url参数来传参，我们应该用`context.Query(xxx)`来获取参数。**
+ **API调用完成后我们返回的是JSON数据，根据不同的需求，我们也可以返回一个渲染好的html页面，或者就返回裸字符串。**
+ **通过`context.JSON()`返回给调用者的是JSON格式的数据，字段就是gin.H{}里的那些字段，gin.H其实只是一个简写，本质上gin.H就是一个`map[string]interface{}。**

## 总结

+ 在gin框架里，中间件指的是在API调用的执行链处于中间位置的组件。那撇开gin框架，从宏观的角度来看，中间件又指什么呢？有哪些著名的中间件？

	+ 关键词：中间件、业务逻辑、插件

+ 用户成功登录后，我们把`JWT`串写到了他的cookie里，对于浏览器应用来说写cookie是不错的做法（虽然可能有安全问题）。那如果调用我们API的不是浏览器应用，而是ios/android app，我们应该把`JWT`串写到哪里呢，应该怎么返回呢？

+ 在返回的JSON数据里，我们有自定义的`code`字段，可是本身http协议就有代表不同状态的状态码，为什么我们还要加上自定义的`code`？

+ 在实现中间件时，`GetXXXMiddleware()`返回的是一个`func(*gin.Context)`，为什么一个函数的返回值可以是另一个函数？在Golang里函数也是一种类型吗？在Java里可不可以实现类似的功能？

    + 关键词：函数指针、函数作为类型

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
