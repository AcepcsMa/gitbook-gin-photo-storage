
## Overview
在上一篇教程 [《[Gin教程-7]用zap替代自带日志库》](https://marcoma.xyz/2019/03/17/gin-tutorial-7/)中我们用`Uber`开源的`zap`日志库替代了Golang自带的默认日志库。至此，“编码”工作真正基本完成。

可是新的问题又来了，在整个编码过程中我们都是自己一个人开发，自己一个人测试，所有的接口参数、调用方式都是自定义的。但假如在一个团队中参与开发，有时候你要依赖于上游的接口，有时候下游要依赖于你写的接口，难道参数和调用方式都要“口口相传”吗？

很明显，我们在编码完成之后，需要用文档的方式记录下所有的API规范，方便他人查阅和使用。但是开发需求很多，编码工作都做不完，哪里有时间写详细的使用文档呢？

基于这个问题，我们迫切需要一个能够帮我们快速生成文档，且其文档易读性较高的工具——`Swagger`。

## Swagger
[Swagger](https://swagger.io/) 的口号是The Best APIs Built with Swagger Tools。它完美地支持`OpenAPI`的语法，利用`OpenAPI`的语法，我们可以编写JSON或者YAML文件，在文件中注明各API的调用方式、参数、返回值、状态码等，最后使用`Swagger`生成可读性高的API文档。

一般来说，有两种使用`Swagger`的套路：

1. 前期API设计就使用`Swagger`，先把接口名、Resource、参数、返回值等全部按照`Swagger`支持的语法写好，然后利用`Swagger Hub`生成测试接口，团队review后才进行真正的编码开发。
2. 前期API设计“随心所欲”，全靠开发者和下游调用团队协商，没有正式规范和文档。等到编码完成&简单的测试完成，才用`Swagger`生成API文档。

可以看出，套路1比较规范，先设计，再按照文档里的约定开发，不太容易出问题，沟通应该也会很顺畅，但是开发周期估计比较长，不适用于赶工上线。套路2比较“野路子”，全靠“口口相传”，先编码保证了能够快速完成需求，但是在联调或者测试过程中可能会出很多bug。

但是由于本教程一开始没有先进行总体的API设计，所以只能采用套路2，等到编码完成了再来生成API文档。**建议大家在实际生产中，不是特别紧急的需求，尽可能地先用`Swagger`设计好API，记录成文档，再进行编码。**

## 代码实现

+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接

    + `go get -u github.com/swaggo/swag/cmd/swag`
    + `go get -u github.com/swaggo/gin-swagger`
    + `go get -u github.com/swaggo/gin-swagger/swaggerFiles`
+ 否则可用go mod来安装

安装完后可执行：

```
swag -version
```
如果安装成功的话会显示该工具的版本号。

然后我们以`AddAuth()`和`GetBucketByID()`作为示例说明如何生成`Swagger`文档：

```golang
// @Summary Add a new auth.
// @version 1.0
// @Accept mpfd
// @Param user_name formData string true "User Name" minlength(6) maxlength(16)
// @Param password formData string true "Password" minlength(6) maxlength(16)
// @Param email formData string true "Email" maxlength(128)
// @Success 200 {string} json "{"code":"","data":{},"msg":"ok"}"
// @Router /api/v1/auth/add [post]
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
			//log.Println(err)
			utils.AppLogger.Info(err.Message, zap.String("service", "AddAuth()"))
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": userName,
		"msg": constant.GetMessage(responseCode),
	})
}
```
如上述代码所示，在`AddAuth()`方法的上方添加注释，非常类似`Java Doc`的写法。

> 参数说明
> 
> @Summary：说明该方法的作用
> 
> @version：方法版本号
> 
> @Accept：`mpfd`意味着该方法接受的请求参数是以`multipart/form data`形式传过来的，对应我们代码里的`context.PostForm("......")`从form data里拿请求参数。除了`mpfd`还有其他取值。
> 
> @Param：定义了具体的请求参数，格式是`param_name param_type data_type is_required description`
> 
> @Success：定义了请求成功执行后的返回值
> 
> @Router：定义了该方法的调用路径，以及调用类型[get、post、put、delete......]

再来看看`GetBucketByID()`方法：

```golang
// @Summary Get bucket by bucket id.
// @version 1.0
// @Param bucket_id query string true "Bucket ID"
// @Success 200 {string} json "{"code":"","data":{},"msg":"ok"}"
// @Router /api/v1/bucket/get_by_id [get]
func GetBucketByID(context *gin.Context) {
	responseCode := constant.INVALID_PARAMS
	bucketID, bucketErr := strconv.Atoi(context.Query("bucket_id"))
	if bucketErr != nil {
		//log.Println(bucketErr)
		utils.AppLogger.Info(bucketErr.Error(), zap.String("service", "GetBucketByID()"))
		context.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
			"code": responseCode,
			"data": make(map[string]string),
			"msg":  constant.GetMessage(responseCode),
		})
		return
	}

	validCheck := validation.Validation{}
	validCheck.Required(bucketID, "bucket_id").Message("Must have bucket id")
	validCheck.Min(bucketID, 1, "bucket_id").Message("Bucket id should be positive")

	data := make(map[string]interface{})
	if !validCheck.HasErrors() {
		if bucket, err := models.GetBucketByID(uint(bucketID)); err != nil {
			if err == models.NoSuchBucketError {
				responseCode = constant.BUCKET_NOT_EXIST
			} else {
				responseCode = constant.INTERNAL_SERVER_ERROR
			}
		} else {
			responseCode = constant.BUCKET_GET_SUCCESS
			data["bucket"] = bucket
		}
	} else {
		for _, err := range validCheck.Errors {
			//log.Println(err.Message)
			utils.AppLogger.Info(err.Message, zap.String("service", "GetBucketByID()"))
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": data,
		"msg":  constant.GetMessage(responseCode),
	})
}
```
同理，注释都是差不多的，差别只在于这个API需要用`get`调用。

参照上述的示例，读者可自行为其他API对应的方法添加`Swagger`注释。

接下来，在代码里添加了注释，要怎么根据注释生成文档呢？于是，我们就得在项目根目录`gin-photo-storage`下执行：

```
swag init
```
通过上述命令初始化`Swagger`文档，初始化成功的话可在`gin-photo-storage`目录下看到一个新的目录：`docs`，里面包含了工具自动生成的文档数据和Golang代码。

为了让用户能够通过一个url来访问`Swagger`文档，我们要为其添加一个Router。修改`gin-photo-storage/routers/router.go`：

```golang
// Init router, adding paths to it.
func init() {
	Router = gin.Default()
	
	// ......

	Router.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// api group for v1
	v1Group := Router.Group("/api/v1")
	{
	   // ......
	}
}
```
通过访问上述添加的路径，在浏览器里即可访问`Swagger`下的所有文档。

**Note：**

+ **如果想改`Swagger`文档内容，在改动方法上面的注释后，还要重新`swag init`一下，以基于新的注释生成新的文档。**
+ **除了上述注释里的参数，`Swagger`还支持很多参数，具体请查阅：[swaggo gitbook](https://swaggo.github.io/swaggo.io/declarative_comments_format/general_api_info.html)。**

## 测试
运行服务端程序，然后通过浏览器访问`localhost:{监听的端口号}/swagger/index.html`，即可看到`Swagger`文档界面：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-swagger-test-1.png)

接下来我们测试一下`AddAuth()`接口，点击**Try it out**：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-swagger-test-2.png)

填入参数，点击**Execute**：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-swagger-test-3.png)

得到执行结果：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-swagger-test-4.png)

从上述截图中可以看到，`Swagger`文档界面里不仅有API的描述和参数说明，还能够让使用者对接口进行简单测试，一举多得。

## 总结

+ 在系列教程里，我们是先写代码，再根据代码生成`Swagger`文档，所以才需要用`gin-swagger`这个工具库。如果想在编码前先设计好API文档，那应该怎么做呢，有固定的语法和文件格式吗？

    + 关键词：Swagger Hub、OpenAPI

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
