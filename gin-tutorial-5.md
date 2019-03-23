
## Overview

在上一篇教程[《[Gin教程-4]中间件与API开发-1》](https://marcoma.xyz/2019/03/04/gin-tutorial-4/)里我们实现了Auth API，可是还没有说怎么可以让它“运行起来”，还不知道怎么可以让用户调用。在本篇教程里，我们要定义后端的Router和Server，还有实现Bucket和Photo的部分API，让我们的服务真正地跑起来，可供外部调用。

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-router.png)

如图所示，所谓的Router，就是路由器，负责把外部请求转发给对应的执行方法。例如用户调用的是`www.xxx.com/add_auth`，服务器接到请求后，Router就把该请求转发给我们上一篇实现的`AddAuth()`方法，执行里面的添加新用户的逻辑。

而Server的实现，通俗来说其实就是要写一些代码，让你的进程一直运行，正常情况下不要自行退出，并且能够监听一个端口，从该端口处能接收用户请求和用户数据。

## Router & Main
因为上一篇教程里我们已经实现了Auth API，所以可以先简单地实现Router和Server，试试我们的API能不能正常工作。

在项目的根目录下新建`routers`文件夹，然后在文件夹内新建`router.go`代码文件：

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
    |—— ......
|—— routers
    |—— router.go
```
Router的逻辑无非就是新建一个Router对象，然后往该Router对象里添加各种路径 -> 方法的映射关系。

```golang
package routers

import (
	"gin-photo-storage/apis/v1"
	"gin-photo-storage/middleware"
	"github.com/gin-gonic/gin"
)

// a global router
var Router *gin.Engine

// Init router, adding paths to it.
func init() {
	Router = gin.Default()
	
	// api group for v1
	v1Group := Router.Group("/api/v1")
	{
		// api group for authentication
		authGroup := v1Group.Group("/auth")
		{
			authGroup.POST("/add", v1.AddAuth)
			authGroup.POST("/check", v1.CheckAuth)
		}
	}
}
```
**Note：**

+ **用Group来划分API，可以让组织结构更加清晰。例如第一版的API，我们全部把它们归到`/api/v1`这个group下面，当项目迭代要开发第二版时，可以另外开一个group叫`/api/v2`，避免冲突。当调用该group下的API时，路径都要带上该group的前缀，如`www.xxx.com/api/v1/......`。**
+ **Group是可以嵌套的，在v1的group内部，基于v1还可以按照业务或者Model来划分不同的group，可以有`authGroup`、`bucketGroup`、`photoGroup`......同理，调用的时候也要加上该group的路径前缀。**

接下来在项目的根目录下直接新建`main.go`代码文件：

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
    |—— ......
|—— routers
    |—— ......
|—— main.go
```
在Main里面，主要的逻辑就是起一个http server，然后配置一下该http server的handler，让它把请求转交给我们的Router去处理。

```golang
package main

import (
	"fmt"
	"gin-photo-storage/conf"
	"gin-photo-storage/routers"
	"gin-photo-storage/constant"
	"net/http"
)

func main() {
    // get the global router from router.go
	router := routers.Router

    // set up a http server
	server := http.Server{
		Addr: fmt.Sprintf(":%s", conf.ServerCfg.Get(constant.SERVER_PORT)),
		Handler: router,
		MaxHeaderBytes: 1 << 20,
	}

    // run the server
	server.ListenAndServe()
}
```
最后，**保证你的`MySQL`和`Redis`在正常运行**，然后在项目根目录下执行`go run main.go`，http server就能正常跑起来，保持监听我们在`server.conf`里设定的那个端口。

既然已经跑起来了，我们可以试试调用`AddAuth()`，看看是不是能够成功写入数据库。**此处强烈建议使用`Postman`工具来测试API调用。（可以在电脑上装Postman软件，也可以用它的chrome插件）**

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-postman.png)

用`Postman`调用`add_auth`API。（我设的监听端口是9088，测试时请更换为你们自己的端口）

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-db-addauth.png)

可以看到在数据库的`auth`表里出现了一条新增的用户记录。

## Bucket API
在`gin-photo-storage/apis/v1/`下新建`bucket.go`代码文件：

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
        |—— bucket.go
|—— routers
    |—— ......
|—— main.go
```
然后为Bucket实现增删改查的API，**因为篇幅和排版的原因，教程里只给出`AddBucket()`的源码。**其他API的代码逻辑基本类似，没有太大区别，全部Bucket API的源码可参考github repo里的[bucket.go](https://github.com/AcepcsMa/gin-photo-gallery-example/blob/master/apis/v1/bucket.go)

```golang
package v1

import (
	"gin-photo-storage/models"
	"gin-photo-storage/constant"
	"github.com/astaxie/beego/validation"
	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"log"
	"net/http"
	"strconv"
)

// Add a new bucket.
func AddBucket(context *gin.Context) {
	responseCode := constant.INVALID_PARAMS
	bucketToAdd := models.Bucket{}
	if err := context.ShouldBindWith(&bucketToAdd, binding.Form); err != nil {
		log.Println(err)
		context.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
			"code": responseCode,
			"data": make(map[string]string),
			"msg":  constant.GetMessage(responseCode),
		})
		return
	}

	validCheck := validation.Validation{}
	validCheck.Required(bucketToAdd.AuthID, "auth_id").Message("Must have auth id")
	validCheck.Required(bucketToAdd.Name, "bucket_name").Message("Must have bucket name")
	validCheck.MaxSize(bucketToAdd.Name, 64, "bucket_name").Message("Bucket name length can not exceed 64")

	if !validCheck.HasErrors() {
		if err := models.AddBucket(&bucketToAdd); err != nil {
			if err == models.BucketExistsError {
				responseCode = constant.BUCKET_ALREADY_EXIST
			} else {
				responseCode = constant.INTERNAL_SERVER_ERROR
			}
		} else {
			responseCode = constant.BUCKET_ADD_SUCCESS
		}
	} else {
		for _, err := range validCheck.Errors {
			log.Println(err.Message)
		}
	}

	data := make(map[string]string)
	data["bucket_name"] = bucketToAdd.Name

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": data,
		"msg":  constant.GetMessage(responseCode),
	})
}

// Delete an existed bucket.
func DeleteBucket(context *gin.Context) {
	// ......
}

// Update an existed bucket.
func UpdateBucket(context *gin.Context) {
	// ......
}

// Get a bucket by bucket id.
func GetBucketByID(context *gin.Context) {
	// ......
}

// Get buckets by auth id.
func GetBucketByAuthID(context *gin.Context) {
	// ......
}
```
**Note：**

+ **这里没有用`context.Form()`或者`context.Query()`来获取请求里的参数，而是直接把请求参数bind到一个Bucket对象上（`context.ShouldBindWith()`）。为什么可以这么做呢？原因就是我们在Bucket Model里为每个字段设定了`form`标签，所以只要请求参数里的参数名能严格对应字段的`form`标签，就能成功把参数绑定到Bucket Model对象上。**

## Photo API
在`gin-photo-storage/apis/v1/`下新建`photo.go`代码文件：

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
        |—— bucket.go
        |—— photo.go
|—— routers
    |—— ......
|—— main.go
```
然后为Photo实现增删改查的API，**因为篇幅和排版的原因，教程里只给出`AddPhoto()`的源码。**其他API的代码逻辑基本类似，没有太大区别，全部Photo API的源码可参考github repo里的[photo.go](https://github.com/AcepcsMa/gin-photo-gallery-example/blob/master/apis/v1/photo.go)

```golang
package v1

import (
	"gin-photo-storage/models"
	"gin-photo-storage/constant"
	"github.com/astaxie/beego/validation"
	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"log"
	"net/http"
	"strconv"
	"strings"
)

// Add a new photo
func AddPhoto(context *gin.Context) {
	responseCode := constant.INVALID_PARAMS

	photoFile, fileErr := context.FormFile("photo")
	if fileErr != nil {
		log.Println(fileErr)
	}

	photo := models.Photo{}
	paramErr := context.ShouldBindWith(&photo, binding.Form)

	if fileErr != nil || paramErr != nil {
		context.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
			"code": responseCode,
			"data": make(map[string]string),
			"msg":  constant.GetMessage(responseCode),
		})
		return
	}

	validCheck := validation.Validation{}
	validCheck.Required(photo.AuthID, "auth_id").Message("Must have auth id")
	validCheck.Required(photo.BucketID, "bucket_id").Message("Must have bucket id")
	validCheck.Required(photo.Name, "photo_name").Message("Must have photo name")
	validCheck.MaxSize(photo.Name, 255, "photo_name").Message("Photo name len must not exceed 255")

	data := make(map[string]interface{})
	photoToAdd := &models.Photo{BucketID: photo.BucketID, AuthID: photo.AuthID,
		Name: photo.Name, Description: photo.Description,
		Tag:strings.Join(photo.Tags, ";")}

	if !validCheck.HasErrors() {
		if photoToAdd, uploadID, err := models.AddPhoto(photoToAdd, photoFile); err != nil {
			if err == models.PhotoExistsError {
				responseCode = constant.PHOTO_ALREADY_EXIST
			} else {
				responseCode = constant.INTERNAL_SERVER_ERROR
			}
		} else {
			responseCode = constant.PHOTO_ADD_IN_PROCESS
			data["photo"] = *photoToAdd
			data["photo_upload_id"] = uploadID
		}
	} else {
		for _, err := range validCheck.Errors {
			log.Println(err.Message)
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": data,
		"msg":  constant.GetMessage(responseCode),
	})
}

// Delete an existed photo.
func DeletePhoto(context *gin.Context) {
	// ......
}

// Update an existed photo.
func UpdatePhoto(context *gin.Context) {
	// ......
}

// Get a photo by photo id.
func GetPhotoByID(context *gin.Context) {
	// ......
}

// Get photos by bucket id.
func GetPhotoByBucketID(context *gin.Context) {
	// ......
}

// Get the upload status of a photo by upload id.
func GetPhotoUploadStatus(context *gin.Context) {
	// ......
}
```
同理，除了用`context.ShouldBindWith()`来绑定参数之外，我们还用`context.FormFile()`来获取用户通过API上传的图片文件。

## 设置Router转发规则
到这里，我们已经实现了：

+ 中间件
    + pagination中间件
    + auth中间件
    + refresh中间件
+ API
    + Auth API
    + Bucket API
    + Photo API

此时应该把所有API的routing规则都设置到Router里，让请求真正都能被转发到对应的执行方法处。以下为最终的`router.go`：

```golang
package routers

import (
	"gin-photo-storage/apis/v1"
	"gin-photo-storage/middleware"
	"github.com/gin-gonic/gin"
)

// a global router
var Router *gin.Engine

// Init router, adding paths to it.
func init() {
	Router = gin.Default()
	checkAuthMdw := middleware.GetAuthMiddleware()			// middleware for authentication
	refreshMdw := middleware.GetRefreshMiddleware()			// middleware for refresh auth token
	paginationMdw := middleware.GetPaginationMiddleware()	// middleware for pagination

	// api group for v1
	v1Group := Router.Group("/api/v1")
	{
		// api group for authentication
		authGroup := v1Group.Group("/auth")
		{
			authGroup.POST("/add", v1.AddAuth)
			authGroup.POST("/check", v1.CheckAuth)
		}

		// api group for bucket
		bucketGroup := v1Group.Group("/bucket")
		{
			// must check auth & refresh auth token before any operation
			bucketGroup.POST("/add", checkAuthMdw, refreshMdw, v1.AddBucket)
			bucketGroup.DELETE("/delete", checkAuthMdw, refreshMdw, v1.DeleteBucket)
			bucketGroup.PUT("/update", checkAuthMdw, refreshMdw, v1.UpdateBucket)
			bucketGroup.GET("/get_by_id", checkAuthMdw, refreshMdw, v1.GetBucketByID)
			bucketGroup.GET("/get_by_auth_id", checkAuthMdw, refreshMdw, paginationMdw, v1.GetBucketByAuthID)
		}

		// api group for photo
		photoGroup := v1Group.Group("/photo")
		{
			// must check auth & refresh auth token before any operation
			photoGroup.POST("/add", checkAuthMdw, refreshMdw, v1.AddPhoto)
			photoGroup.GET("/upload_status", checkAuthMdw, refreshMdw, v1.GetPhotoUploadStatus)
			photoGroup.DELETE("/delete", checkAuthMdw, refreshMdw, v1.DeletePhoto)
			photoGroup.PUT("/update", checkAuthMdw, refreshMdw, v1.UpdatePhoto)
			photoGroup.GET("/get_by_id", checkAuthMdw, refreshMdw, v1.GetPhotoByID)
			photoGroup.GET("/get_by_bucket_id", checkAuthMdw, refreshMdw, paginationMdw, v1.GetPhotoByBucketID)
		}
	}
}
```
可以看到，在bucket group和photo group里我们为每个API都设定了调用路径，并且为其配置了auth中间件、pagination中间件和refresh中间件。

## 总结

+ Router定义里，不同的方法会对应`GET`、`POST`、`PUT`、`DELETE`多个不同的动词，它们有什么区别？除了这4个还有什么动词可以直接拿来用？

    + 关键词：http verb

+ 在教程里，Server、Router、APIs、Models全都是一个进程里的，在单台服务器上把程序run起来就可以工作了。但如果程序里某一个地方出bug导致崩溃了，意味着从Server到Models都不能工作了，有更好的架构解决类似的问题吗？

    + 关键词：微服务、容灾

+ 我们在main里面起了一个http server，什么是http？可以用http server来做即时聊天工具的服务器吗？

    + 关键词：http协议、tcp协议、网络协议栈

+ 在`AddAuth()`的逻辑里，我们会先把用户密码用md5加一层壳再写MySQL，但是在调API的时候是直接明文传输的，这样是不是很不安全？有什么解决手段吗？

    + 关键词：SSL/TLS、https

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
