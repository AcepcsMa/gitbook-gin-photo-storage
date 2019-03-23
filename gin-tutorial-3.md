
## Overview

上一篇[《[Gin教程-2]数据库与Models实现》](https://marcoma.xyz/2019/02/25/gin-tutorial-2/)里设计了MySQL表以及对应的Model，并且为每个Model都实现了一些必要的增/删/改/查接口。利用MySQL，我们可以把很多数据都持久化下来。但是在本项目中也有一些数据是MySQL不擅长处理的，或者说有别的存储组件比MySQL更适合用来实现我们的需求，如：

1. 记录用户登录状态

	用户成功登录后系统要记录Ta已登录；登录的有效期为X秒，X秒内如果没有任何操作，X秒后登录状态要过期，要求用户重新登录；用户可主动退出登录，系统要及时更新Ta的登录状态。**更新操作非常频繁，不是那种一次写多次读的数据。**
2. 当做伪消息队列，在多个模块间传递消息

	当一个操作不是原子操作，而是由多个小步骤组合而成，那么多个小步骤的执行顺序很可能需要通过消息来驱动。例如本项目的添加图片功能，第一步就是早MySQL表里插入新record，第二步就是将图片上传至腾讯云的对象存储，第三步就是将腾讯云的图片url更新到MySQL表中。第二和第三步之间就要通过消息来驱动，只有图片上传成功后，才能通知把图片url更新到表里。
3. 存储图片数据

	图片实际上就是一连串的字符数据，如果我们想把图片存下来，可以把这一串字符串存到MySQL里的一个text字段，又或者把图片存到本地的文件系统里，然后MySQL里记录该图片的文件路径。但是其实有更好的办法，就是用一些公有云提供的对象存储服务来存图片、视频、音频等非结构化的数据。
	
针对1、2的需求，我们可以用`Redis`来实现。**本教程不涉及安装`Redis`的步骤，请自行查阅并安装。**

针对3的需求，我们可以用`七牛云`/`阿里云oss`/`aws S3`/`腾讯云cos`来实现。**由于`腾讯云cos`有50G的免费额度，基于这个原因本教程就选用了`腾讯云cos`**，实际上各家的对象存储服务在简单存储上都没有太明显的差异，可根据自己的喜好选择。


## 实现Redis Utils
首先在`utils`下新建`redis.go`代码文件。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
    |—— jwt.go
	|—— redis.go
```
本教程使用的redis client是`go-redis`，当然github上还有很多不错的redis client，也可以根据个人喜好更换。

+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u github.com/go-redis/redis`
+ 否则可用go mod来安装

根据上面提到的需求，我们要用`Redis`实现以下接口：

1. 用户登录后，把用户信息添加到`Redis`里，并设置过期时间
2. 查询用户信息是否在`Redis`里（是否登录成功）
3. 用户退出登录后，把用户信息从`Redis`里删除
4. 图片开始上传时，把图片信息添加到`Redis`里
5. 查询图片上传状态
6. 消息通知，把消息发布到某个`Redis channel`

先在`gin-photo-storage/constant/constant.go`里加入几个常量：

```golang
// Redis constants
REDIS_HOST = "REDIS_HOST"
REDIS_PORT = "REDIS_PORT"
```
然后记得在`gin-photo-storage/conf/server.conf`里写上`Redis`的相关配置项。没有自行修改的话，默认地址是本机地址`127.0.0.1`，默认端口是`6379`。

```
{
	......
	"REDIS_HOST": "127.0.0.1",
	"REDIS_PORT": "6379",
	......
}
```
接下来开始实现`redis.go`。

```golang
// 以下为redis.go的内容
package utils

import (
	"fmt"
	"gin-photo-storage/conf"
	"gin-photo-storage/constant"
	"github.com/go-redis/redis"
	"log"
	"strconv"
	"time"
)

var RedisClient *redis.Client
var InitComplete = make(chan struct{}, 1)

// Init redis client
func init() {
	host := conf.ServerCfg.Get(constant.REDIS_HOST)
	port := conf.ServerCfg.Get(constant.REDIS_PORT)
	RedisClient = redis.NewClient(&redis.Options{
		Addr: fmt.Sprintf("%s:%s", host, port),
		Password: "",
		DB: 0,
	})
	InitComplete <- struct{}{}
}

// Add an auth to redis, meaning that he/she has logged in.
func AddAuthToRedis(username string) error {
	key := fmt.Sprintf("%s%s", constant.LOGIN_USER, username)
	err := RedisClient.Set(key, username, constant.LOGIN_MAX_AGE * time.Second).Err()
	if err != nil {
		log.Fatalln(err)
		return err
	}
	return nil
}

// Check if an auth is in redis.
func IsAuthInRedis(username string) bool {
	key := fmt.Sprintf("%s%s", constant.LOGIN_USER, username)
	err := RedisClient.Get(key).Err()
	if err != nil {
		log.Println(err)
		return false
	}
	return true
}

// Remove an auth from redis, meaning he/she is logging out.
func RemoveAuthFromRedis(username string) bool {
	key := fmt.Sprintf("%s%s", constant.LOGIN_USER, username)
	err := RedisClient.Del(key).Err()
	if err != nil {
		log.Println(err)
		return false
	}
	return true
}

// Set the upload status for a photo.
func SetUploadStatus(key string, value int) bool {
	err := RedisClient.Set(key, value, 0).Err()
	if err != nil {
		log.Println(err)
		return false
	}
	return true
}

// Get the upload status of a photo.
func GetUploadStatus(key string) int {
	val := RedisClient.Get(key).Val()
	if val == "" {
		return -2	// means no such key
	}
	status, _ := strconv.Atoi(val)
	return status
}

// Send a message to the given channel.
func SendToChannel(channel string, message string) bool {
	err := RedisClient.Publish(channel, message).Err()
	if err != nil {
		log.Println(err)
		return false
	}
	return true
}
```
这里有几点核心点需要注意：

1. 在`init()`里，初始化连接后，往`InitComplete`这个channel发了一条空数据，主要是有别的模块一直监听这个channel，阻塞在那，等待Redis连接初始化的完成。通过这条空数据可以告诉别的模块“我已经把Redis连接初始化好了，你们可以开始使用该连接了”。
2. 用户登录后往Redis里写信息，key是`LOGIN_ + 用户名`，value是用户名，且设了一个过期时间`constant.LOGIN_MAX_AGE`，查询和删除用户信息也是用的相同的key。
3. 发消息不是真正意义上的点对点发消息，而是用了Redis本身的订阅发布（pub-sub）机制：`Publish(channel, message)`。

## 实现COS Utils
**Note：首先注册腾讯云账号（应该用微信账号即可），确保开通了腾讯云COS，并且为本项目新建了一个bucket，bucket名称任意。**

首先在`utils`下新建`cos.go`代码文件。

```
gin-photo-storage
|—— conf
 	|—— ......
|—— constant
	|—— ......
|—— models
	|—— ......
|—— utils
    |—— jwt.go
	|—— redis.go
	|—— cos.go
```
+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u github.com/tencentyun/cos-go-sdk-v5`
+ 否则可用go mod来安装

先在`gin-photo-storage/constant/constant.go`里加入几个常量：

```golang
// COS constants
COS_BUCKET_NAME = "COS_BUCKET_NAME"
COS_APP_ID 		= "COS_APP_ID"
COS_REGION 		= "COS_REGION"
COS_SECRET_ID 	= "COS_SECRET_ID"
COS_SECRET_KEY 	= "COS_SECRET_KEY"
```
然后记得在`gin-photo-storage/conf/server.conf`里写上`COS`的相关配置项。以下的配置项均可在腾讯云的控制面板里获取。

```
{
	......
	"COS_SECRET_ID": "......",
	"COS_SECRET_KEY": "......",
	"COS_BUCKET_NAME": "......",
	"COS_APP_ID": "......",
	"COS_REGION": "......",
	......
}
```
接下来开始实现`cos.go`。

```golang
// 以下为cos.go的内容
package utils

import (
	"context"
	"fmt"
	"gin-photo-storage/conf"
	"gin-photo-storage/constant"
	"github.com/tencentyun/cos-go-sdk-v5"
	"io"
	"log"
	"net/http"
	"net/url"
)

var (
	CosClient    *cos.Client
	CosUrlFormat = "http://%s-%s.cos.%s.myqcloud.com"
	BucketName   = ""
	AppID        = ""
	Region       = ""
)

// init COS client
func init() {
	BucketName = conf.ServerCfg.Get(constant.COS_BUCKET_NAME)
	AppID = conf.ServerCfg.Get(constant.COS_APP_ID)
	Region = conf.ServerCfg.Get(constant.COS_REGION)
	u, _ := url.Parse(fmt.Sprintf(CosUrlFormat, BucketName, AppID, Region))
	b := &cos.BaseURL{BucketURL: u}
	CosClient = cos.NewClient(b, &http.Client{
		Transport: &cos.AuthorizationTransport{
			SecretID:  conf.ServerCfg.Get(constant.COS_SECRET_ID),
			SecretKey: conf.ServerCfg.Get(constant.COS_SECRET_KEY),
		},
	})
	log.Printf("COS client %s init", CosClient.BaseURL)
}
```
## 结合Redis和COS实现异步图片上传
到了这一步，已经把`Redis utils`和`COS utils`实现后，**可以把它们结合起来实现上一篇教程遗留的一个需求——图片上传。**

我们有两种方式可以实现图片上传：

+ 同步（sync）上传。调用AddPhoto后，先插入一条数据库record，再上传图片至COS，等待图片上传完成，更新MySQL record的url字段，最后才返回给调用者。
+ 异步（async）上传。调用AddPhoto后，先插入一条数据库record，马上返回给调用者，状态为“上传中”。同时启动一个goroutine在后台执行上传操作，上传完成后再执行回调（callback）函数来更新MySQL record的url字段。

这两种方式的优劣非常明显。同步上传很容易实现且出错了很容易处理（要么立刻重试，要么放弃），但是需要调用者在调用AddPhoto后长时间等待（取决于网速/COS的速度/COS的稳定性）；异步上传则比较复杂，涉及组件间的通信（回调），但是API返回速度快，调用者无需等待图片上传。

再结合业务层面想一想哪种方式比较好。当一个用户点击新增图片后，是让用户在当前页面等3~5秒钟，当前页面不断转圈圈等待返回？还是让用户马上返回，在页面的某处显示一个进度框提示上传状态比较好？

显然，异步上传比同步上传更加“人性化”，所以我们就来借助`Redis`和goroutine实现异步上传。

具体的设计如下图所示：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-async-upload.png)

**Note: 在第4步启动后台goroutine后，并不需要等待4.1和4.2完成才执行第5步。而是启动了后台goroutine后马上就能返回给用户（5 & 6），goroutine会在后台完成4.1和4.2步，不存在阻塞等待的过程。**

基于以上的设计，在代码层面我们要实现：

1. callback监听模块（时刻监听着`Redis`里的某个channel，监听到新消息后立即更新MySQL）
2. 往COS上传图片的方法
3. 新建goroutine，调用上传图片的方法

首先实现callback监听模块。在`gin-photo-storage/constant/constant.go`里先加上几个常量。

```golang
// Callback constants
URL_UPDATE_CHANNEL 		= "PHOTO_URL_UPDATE"
PHOTO_UPDATE_ID_FORMAT 	= "photo-%d"
PHOTO_DELETE_CHANNEL 	= "PHOTO_DELETE"
```
因为callback主要是针对MySQL做更新操作，所以把逻辑写在`db.go`里。

```golang
// 在db.go里补充以下代码

// Init the database connection.
func init() {
	// ......
	go ListenRedisCallback()	// launch a background goroutine to listen to callbacks from redis
}

// Listen to callback messages from redis channels.
// 1. When a photo is uploaded successfully, the callback asks to update the photo url in the db.
// 2. When it fails to upload a photo, the callback asks to delete the photo record in the db.
func ListenRedisCallback() {

	// wait until utils package is initialized
	<- utils.InitComplete

	// subscribe redis channels
	updateChan := utils.RedisClient.Subscribe(constant.URL_UPDATE_CHANNEL).Channel()
	deleteChan := utils.RedisClient.Subscribe(constant.PHOTO_DELETE_CHANNEL).Channel()

	// loop and listen
	for {
		select {
		case msg := <-updateChan:
			photoID, _ := strconv.Atoi(msg.Payload[:strings.Index(msg.Payload, "-")])
			photoUrl := msg.Payload[strings.Index(msg.Payload, "-") + 1:]
			if err := UpdatePhotoUrl(uint(photoID), photoUrl); err != nil {
				log.Println(err)
			} else {
				utils.SetUploadStatus(fmt.Sprintf(constant.PHOTO_UPDATE_ID_FORMAT, photoID), 0)
			}
		case msg := <- deleteChan:
			photoID, _ := strconv.Atoi(msg.Payload)
			if err := DeletePhotoByID(uint(photoID)); err != nil {
				log.Println(err)
			} else {
				utils.SetUploadStatus(fmt.Sprintf(constant.PHOTO_UPDATE_ID_FORMAT, photoID), -1)
			}
		default:
		}
	}
}
```
此处的核心在于，当`db.go`所属的package（`models`）被加载时，在`Init()`方法的最后会启动一个后台goroutine，该goroutine实际上就是一个无限的for循环，订阅了`Redis`里的更新图片url的channel、删除图片的channel，即一直保持监听。在channel里接收到新消息后，就去执行“更新MySQL里的图片url”或者“删除图片”。

接着利用`COS utils`实现上传图片，即在`cos.go`里实现`Upload()`和`AsyncUpload()`。

```golang
// 以下代码在cos.go里实现

// upload a photo to the tencent cloud COS
// @photoID: 图片id
// @fileName: 图片名
// @file: 图片句柄
// @fileSize: 图片大小
func Upload(photoID uint, fileName string, file io.Reader, fileSize int) string {
	uploadID := fmt.Sprintf(constant.PHOTO_UPDATE_ID_FORMAT, photoID)
	go AsyncUpload(uploadID, photoID, fileName, file, fileSize)	// upload in the ASYNC way
	return uploadID
}

// upload a photo to the tencent cloud COS in the ASYNC way
func AsyncUpload(uploadID string, photoID uint, fileName string, file io.Reader, fileSize int) {
	// set upload status in redis
	if !SetUploadStatus(uploadID, 1) {
		log.Println("Fail to set upload status before upload.")
		return
	}

	// upload the photo using COS SDK
	putOption := cos.ObjectPutOptions{}
	putOption.ObjectPutHeaderOptions = &cos.ObjectPutHeaderOptions{ContentLength: fileSize}
	_, err := CosClient.Object.Put(context.Background(), fileName, file, &putOption)

	// upload fails, send callback asking for photo deletion
	if err != nil {
		log.Println(err)
		if !SendToChannel(constant.PHOTO_DELETE_CHANNEL, fmt.Sprintf("%d", photoID)) {
			log.Println("Fail to send delete-photo message to channel")
		}
		return
	}

	// upload success, send callback asking for updating the photo url
	fileUrl := fmt.Sprintf(CosUrlFormat, BucketName, AppID, Region) + "/" + fileName
	updateUrlMessage := fmt.Sprintf("%d-%s", photoID, fileUrl)
	if !SendToChannel(constant.URL_UPDATE_CHANNEL, updateUrlMessage) {
		log.Println("Fail to send update-photo-url message to channel")
	}
}
```
可以看到，**在`Upload()`里直接另起了一个goroutine来执行`AsyncUpload()`，然后不需要等上传完成就马上返回了，所以才能达到“异步”的效果。**当`AsyncUpload()`里完成了图片上传，上传成功就往`Redis`里面的`URL_UPDATE_CHANNEL`里发消息，失败就往`PHOTO_DELETE_CHANNEL`里发消息，该消息会被我们上面实现的callback模块监听到，由callback来执行后面的更新或删除逻辑。

最后，因为上一篇教程里在`AddPhoto()`里没有上传图片的代码，所以别忘了在`AddPhoto()`里加上调用图片上传的代码。

```golang
// 以下代码在photo.go里实现

// Add a new photo
func AddPhoto(photoToAdd *Photo, photoFileHeader *multipart.FileHeader) (*Photo, string, error) {
	// ......
	// upload to the tencent cloud COS
	if photoFile, err := photoFileHeader.Open(); err == nil {
		uploadID := utils.Upload(photo.ID, photo.Name, bufio.NewReader(photoFile), int(photoFileHeader.Size))
		return &photo, uploadID, nil
	} else {
		log.Println(err)
		return nil, "", PhotoFileBrokenError
	}
}
```

## Others
本篇教程到这里，`Redis utils`、`COS utils`和图片异步上传已经全部实现了。可能有的读者会觉得怪怪的，如果我们用异步的方式上传图片，调用者确实不需要等待，马上能得到一个“上传中”的response，但它要怎么知道上传到底有没有完成，还有是什么时候完成的呢？

注意看`AsyncUpload()`和`ListenRedisCallback()`里，都有执行`SetUploadStatus()`。上传之前，都先把图片的upload status设成1。如果成功上传，在`ListenRedisCallback()`里会将其设为0；如果上传失败，`ListenRedisCallback()`里会将其设为-1。

再留意`redis.go`，我们还实现了一个`GetUploadStatus()`，通过这个接口，就可以查询某张图片的上传状态。从前端实现的角度而言，想要知道图片上传成功与否，每隔N秒调用一下这个接口，询问上传状态即可。

## 总结

+ 这里把`Redis`当做伪消息队列来用，利用的是它的pub-sub机制，那么pub-sub的底层是怎么实现的？

	+ 关键词：`Redis`源码、订阅发布

+ 用真正的消息队列组件来实现通知，如RocketMQ、Kafka等，高并发下可能会有什么问题？

	+ 关键词：消息队列、exactly once、幂等操作

+ 在`redis.go`里，`InitComplete = make(chan struct{}, 1)`，为什么要加一个size参数1？不加size参数会生成什么样的channel？他们有什么异同？

	+ 关键词：Golang缓冲通道、阻塞

+ 异步加载看起来很“先进”，不需要阻塞等待，但是潜在的风险也不少。如果一个接口不止对应一个异步操作（例如不只是上传，可能包括了10个独立的异步操作），那假如其中一个操作失败了，怎么处理这个失败？是应该不断重试？还是应该全部步骤都回滚？假如说3号步骤失败了，怎么通知1、2、4~10号步骤都回滚？

	+ 关键词：失败补偿、异步操作

+ 异步上传的核心，除了通过`Redis`做消息传递实现回调外，更重要的是goroutine不会让当前线程阻塞，自己在后台就能执行逻辑。goroutine看着很像Java、Python、C++里的Thread，那goroutine和其他语言里的线程有什么异同，和内核态的线程又是什么关系？

	+ 关键词：goroutine、协程、线程

+ 我们实现的上传逻辑，是让调用者先把一张图片的数据全部post到服务端，然后再从服务器上传到腾讯云COS的。这样无疑很浪费服务器的带宽，假如一张图片5MB，要从客户端接收5MB，再往COS发送5MB，高并发下绝对带宽不足，有什么方法可以改进吗？

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
