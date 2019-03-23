
## Overview
在上一篇教程 [《[Gin教程-6]elasticsearch实现搜索需求》](https://marcoma.xyz/2019/03/10/gin-tutorial-6/)中我们已经把最后一个“业务逻辑”实现完了。按理说从代码的角度没有什么好写了，但是有一个细节可能部分读者会注意到，那就是日志模块——`log`的实现。

在之前的代码里，我们在很多地方写过日志：在操作`MySQL`、`Redis`、`Elasticsearch`的时候要打日志，在API里面处理请求参数出错要打日志，在中间件执行时要打日志......

回看代码，我们用的日志模块都是Golang自带的`log`。通过`log`模块输出的日志都是“raw log”，即一个简单的字符串，如：

```
2019-03-18 02:12:03,822 INFO xx/yy/zz.go hello world
```
这样的日志是最“朴素”的，有不少缺点。首先它对人而言可读性不高，全靠空格分隔，当msg里有大量空格分隔的单词时很难阅读。另外，日志里只有“值”，并没有字段名，假如一条日志里有50个字段，那读着读着就不知道自己读到哪个字段了。再者，如果要对这种“raw log”进行数据分析，写解析程序的时候也比较麻烦，相当于要手动操作字符串：切分、trim、用index取字段......

既然“raw log”有这么多不便之处，我们应该使用更友好、更高效的日志格式，最广泛使用的就是`JSON`。我们可以把上面的那条“raw log”改成成JSON格式，类似于：

```
{
    "time": "2019-03-18 02:12:03,822",
    "level": "INFO",
    "caller": "xx/yy/zz.go",
    "msg": "hello world",
    ......
}
```
如果每一条日志都能输出为这样的`JSON`串，那既提高了可读性，又方便后续的日志分析（读取字段时直接按字段名来读，不用手动操作字符串）。

所以，在本篇教程中我们要引入`zap`这个第三方日志库，来实现日志的`JSON`格式化输出。

## zap
[zap](https://github.com/uber-go/zap) 是`Uber`开源的一个日志库，选择`zap`的原因有3个：

+ `zap`原生支持JSON格式的日志输出
+ `zap`的性能在多个第三方日志库中名列前茅（根据其README里声明的benchmark）
+ `zap`的API设计比较清晰，对用户友好

另外，在Github上比较著名的Golang日志库还有：`zerolog`、`logrus`等，有兴趣的读者可查阅相关资料，自行对比多个日志库。

## 代码实现
首先我们需要把对应的package导入到本地

+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u go.uber.org/zap`
+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u gopkg.in/natefinch/lumberjack.v2`
+ 否则可用go mod来安装

首先在`utils`目录下新建`logging.go`代码文件：

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
	|—— logging.go
|—— middleware
	|—— ......
|—— apis
    |—— ......
|—— routers
    |—— ......
|—— main.go
```
然后在`logging.go`里自定义一个zap logger：

```golang
package utils

import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
	"gopkg.in/natefinch/lumberjack.v2"
)

var AppLogger *zap.Logger

func init() {
	writer := zapcore.AddSync(&lumberjack.Logger{
		Filename:   "logs/app.log",
		MaxSize:    100,
		MaxBackups: 3,
		MaxAge:     1,
	})

	encoderConfig := zapcore.EncoderConfig{
		TimeKey:        "time",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
		EncodeTime:     zapcore.ISO8601TimeEncoder,
		EncodeDuration: zapcore.SecondsDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}

	core := zapcore.NewCore(
		zapcore.NewJSONEncoder(encoderConfig),
		writer,
		zap.InfoLevel,
	)

	caller := zap.AddCaller()
	AppLogger = zap.New(core, caller)
}
```
**Note：**

+ **为了支持滚动日志（Log Rotation），此处用lumberjack里的logger作为我们的底层logger，因为zap本身不支持滚动日志的特性。**
+ **使用`JSONEncoder`来输出JSON格式的日志，而非“raw log string”。**
+ **在`editorConfig`里我们对logger进行设置，设定了一些字段名，还有字段的格式，例如日志level用小写，时间格式用`ISO8601`等。这些配置项均可自由设置，不是一定要用上述代码的设置。**

实现了自定义的logger后，我们就要把之前代码里的`log.Println()`全都改成使用自定义logger来打log。例如在`AddAuth()`中：

```golang
func AddAuth(context *gin.Context) {

	// ......
	
	if !validCheck.HasErrors() {
		// ......
	} else {
		for _, err := range validCheck.Errors {
			utils.AppLogger.Info(err.Message, zap.String("service", "AddAuth()"))
		}
	}

	// ......
}
```
可以看到，除了默认的字段外，我们还加了一个`service`字段，表示这个日志是由哪个API打印出来的。如果读者还有更多需要记录到日志里的字段，也可自行添加。

## Log格式
当我们把所有需要打日志的地方都用自定义的logger输出后，可以来测试一下日志输出。我们使用`Postman`调用`AddAuth()`，故意输入非法的参数，看看日志是否输出JSON格式的错误提醒。

故意设置`user_name`参数小于6位，调用`AddAuth()`：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-zap-test.png)

然后在项目根目录下的`logs`目录里打开`app.log`，看到因非法调用而打印出来的日志：

```
{"level":"info","time":"2019-03-17T23:18:11.809-0700","caller":"v1/auth.go:42","msg":"User name length is at least 6","service":"AddAuth()"}
```
可复制上述日志到 [Json.cn](https://www.json.cn/) 进行验证：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-zap-json-test.png)

## 总结

+ 程序产生的大量日志可以用于作数据分析，那么流行的日志分析工具有哪些呢？

    + 关键词：Elasticsearch、Logstash、Kibana

+ 如果我们的程序并发度很高，意味着写日志的操作并发度也很高，日志最终是要写到磁盘上的日志文件里的，要怎么设计一个写入性能很高的日志库呢？

    + 关键词：buffer、异步、fsync

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
