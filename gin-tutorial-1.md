
## 项目结构
+ 新建项目

	+ Golang版本 <= 1.10，建议直接把项目根目录创在`$GOPATH/src`下
	+ Golang版本 >= 1.11，可以放在本地任意位置，**用go mod来管理第三方package**

+ 目录结构

	```
	# Golang version <= 1.10
	$GOPATH
	|—— src
		|—— gin-photo-storage
			|—— ...
			|—— ...
			|—— ...
	```
	```
	# Golang version >= 1.11
	Anywhere you like
	|—— gin-photo-storage
		|—— ...
		|—— ...
		|—— ...
	```
	对go mod感兴趣的可以去搜索Golang从低版本到高版本一路走来各种package管理的方式。
	
## 如何加载配置文件
首先来看这么几个问题

+ 一个项目里是不是很多地方要用到配置项？

	通常来说，是的。一般项目越大，里面涉及的组件越多（关系型数据库、内存数据库、消息队列......），需要的配置项就越多。你要连数据库总得提供数据库地址&用户名&密码......你要从消息队列收发消息总得提供地址&topic......这些全都是“配置项”。
+ 配置项全部写在代码文件里好不好？

	既然项目里无处不存在“配置项”，那干脆在写业务代码的时候随手一写就好了。但是这样麻烦就大了，假如开发前期用的是开发环境的数据库，地址是`aaa.bbb.ccc.ddd`，到后期上测试了要换成测试环境的数据库，地址是`xxx.yyy.zzz.kkk`，那就要大幅修改N个代码文件，然后提交重新code review。类似的情况远远不止数据库，可以是各种存在多版本问题的配置项。
+ 如果不写在代码文件里，应该写到哪里？

	很明显，如果不写在代码里（所谓的“hard code”），那就只能写在额外的“非代码”文件里，然后你的代码逻辑里有读配置文件的操作。每次要改配置项的值时，改配置文件即可，不需要再改一大堆代码文件。甚至可以为测试环境定义一个`config-test-env`文件，为开发环境定义一个`config-dev-env`文件，为任意环境定义它专属的配置文件，在程序启动时通过命令行参数指定用哪套环境，方便快捷，无痛切换。

所以，整个项目的第一步，我们要新建配置目录`conf`、配置文件：`server.conf`、解析配置文件的代码`cfg.go`。

```
gin-photo-storage
|—— conf
 	|—— server.conf
 	|—— cfg.go
```
在本项目里，直接把配置项写成JSON格式，易读易修改。当然也可以像`EDDYCJY`在[（Gin搭建Blog API's （一））](https://github.com/EDDYCJY/blog/blob/master/golang/gin/2018-02-16-Gin%E5%AE%9E%E8%B7%B5-%E8%BF%9E%E8%BD%BD%E4%BA%8C-%E6%90%AD%E5%BB%BABlogAPIs-01.md)里那样，用`ini`作配置文件的格式，用`ini`对应的package来解析，这个倒是没有太大的讲究。

```
# 以下为server.conf的内容
{
    "JWT_SECRET": "ufqw923dh1nlo9sa",
    "DB_TYPE": "mysql",
    "DB_HOST": "127.0.0.1",
    "DB_PORT": "3306",
    "DB_USER": "......",
    "DB_PWD": "......",
    "DB_NAME": "photo",
    "SERVER_PORT": "9088"
}
```
创建好配置文件后，可以开始写解析配置文件的代码了。因为这里用的JSON格式，所以不需要引用任何第三方package，Golang原生自带JSON序列化&反序列化的库。

```golang
// 以下为cfg.go的代码
package conf

import (
	"encoding/json"
	"log"
	"os"
)

type Cfg struct {
	ConfigMap map[string]string
}

var ServerCfg Cfg

// Init config from the local config file.
func init() {
	confFile, err := os.Open("conf/server.conf")
	defer confFile.Close()
	if err != nil {
		log.Fatalln(err)
	}

	ServerCfg.ConfigMap = make(map[string]string)
	err = json.NewDecoder(confFile).Decode(&ServerCfg.ConfigMap)
	if err != nil {
		log.Fatalln(err)
	}
}

// Get the corresponding config value of the given key.
func (cfg *Cfg) Get(key string) string {
	if val, ok := cfg.ConfigMap[key]; ok {
		return val
	}
	log.Fatalf("No such config term: %s!\n", key)
	return ""
}
```
可以看到，代码里的`init()`就是用来读取同个文件夹下的`server.conf`，然后用原生的JSON解析包对其内容进行解析。另外，为了在其他代码模块中方便读取配置值，我们实现了一个`Get(key string)`方法。

## 常量定义——避免magic number
除了配置项，任何项目里不可缺少的还有各种常量。相比配置项的“动态”而言，常量一般是指跟代码逻辑紧密结合的值，绝大多数情况下基本不会因外部开发环境的变化而变化。不会说切到测试环境就用测试环境的一套，切到开发环境就用开发环境的一套。

例如，针对客户端的请求所回复的数值型状态码（`SUCCESS`、`FAIL`、`PENDING`之类的），或者是代码里的一些常用字符串。这样的值最好还是定义成常量，集中写到一个具体的常量代码文件里。否则直接在N个代码文件里大量使用magic number的话，万一到时要改就比较麻烦。

首先在项目路径下新建`constant`目录，然后在`constant`路径下新建`constant.go`和`resp_code.go`。

```
gin-photo-storage
|—— conf
 	|—— server.conf
 	|—— cfg.go
|—— constant
	|—— constant.go
	|—— resp_code.go
```
在`constant.go`里，我们要定义一些代码里经常使用的字符串/整形常量。

```golang
// 以下为constant.go的内容
package constant

const (
	// JWT constants
	JWT_SECRET 			= "JWT_SECRET"
	JWT 				= "jwt"
	JWT_EXP_MINUTE 		= 30
	PHOTO_STORAGE_ADMIN = "admin"

	// Server constants
	SERVER_PORT = "SERVER_PORT"
	PAGE_SIZE 	= 20

	// DB constants
	DB_CONNECT 	= "%s:%s@tcp(%s:%s)/%s?charset=utf8&parseTime=True&loc=Local"
	DB_TYPE 	= "DB_TYPE"
	DB_HOST 	= "DB_HOST"
	DB_PORT 	= "DB_PORT"
	DB_USER 	= "DB_USER"
	DB_PWD 		= "DB_PWD"
	DB_NAME 	= "DB_NAME"

	// Auth constants
	COOKIE_MAX_AGE 	= 1800
	LOGIN_MAX_AGE 	= 1800
	LOGIN_USER 		= "LOGIN_"
)
```
而在`resp_code.go`里，我们要定义所有API的返回码，毕竟客户端收到回复后基本都要从返回码来判断服务端的执行情况。

```golang
// 以下为resp_code.go的内容
package constant

const (
	// User related responses
	USER_ALREADY_EXIST 		= 1001
	USER_ADD_SUCCESS 		= 1002
	USER_AUTH_SUCCESS 		= 1003
	USER_AUTH_ERROR 		= 1004
	USER_AUTH_TIMEOUT 		= 1005
	USER_SIGNOUT_SUCCESS 	= 1006

	// JWT related responses
	JWT_GENERATION_ERROR 	= 2001
	JWT_MISSING_ERROR 		= 2002
	JWT_PARSE_ERROR 		= 2003

	// Bucket related responses
	BUCKET_ALREADY_EXIST 	= 3001
	BUCKET_ADD_SUCCESS 		= 3002
	BUCKET_NOT_EXIST 		= 3003
	BUCKET_DELETE_SUCCESS 	= 3004
	BUCKET_UPDATE_SUCCESS 	= 3005
	BUCKET_GET_SUCCESS 		= 3006

	// Photo related responses
	PHOTO_ALREADY_EXIST 	= 4001
	PHOTO_ADD_IN_PROCESS 	= 4002
	PHOTO_UPLOAD_SUCCESS 	= 4003
	PHOTO_UPLOAD_ERROR 		= 4004
	PHOTO_NOT_EXIST 		= 4005
	PHOTO_DELETE_SUCCESS 	= 4006
	PHOTO_UPDATE_SUCCESS 	= 4007
	PHOTO_GET_SUCCESS 		= 4008

	// Internal server responses
	INTERNAL_SERVER_ERROR 	= 5001
	PAGINATION_SUCCESS 		= 8001
	INVALID_PARAMS 			= 9001
)

var Message map[int]string

// Init the message map.
func init() {
	Message = make(map[int]string)
	Message[INVALID_PARAMS] 		= "Invalid parameters."
	Message[USER_ALREADY_EXIST] 	= "User already exists."
	Message[USER_ADD_SUCCESS] 		= "Add user success."
	Message[USER_AUTH_SUCCESS] 		= "User authentication success."
	Message[USER_AUTH_ERROR] 		= "User authentication fail."
	Message[USER_AUTH_TIMEOUT] 		= "User authentication timeout."
	Message[USER_SIGNOUT_SUCCESS] 	= "User sign out success."
	Message[JWT_GENERATION_ERROR] 	= "JWT generation fail."
	Message[JWT_MISSING_ERROR] 		= "JWT is missing."
	Message[INTERNAL_SERVER_ERROR] 	= "Internal server error."
	Message[BUCKET_ALREADY_EXIST] 	= "Bucket already exists."
	Message[BUCKET_ADD_SUCCESS] 	= "Add bucket success."
	Message[BUCKET_NOT_EXIST]		= "Bucket does not exist."
	Message[BUCKET_DELETE_SUCCESS] 	= "Bucket delete success."
	Message[BUCKET_UPDATE_SUCCESS] 	= "Bucket update success."
	Message[BUCKET_GET_SUCCESS] 	= "Bucket get success."
	Message[PHOTO_ALREADY_EXIST] 	= "Photo already exists."
	Message[PHOTO_ADD_IN_PROCESS] 	= "Adding photo is in process."
	Message[PHOTO_UPLOAD_SUCCESS] 	= "Photo upload success."
	Message[PHOTO_UPLOAD_ERROR] 	= "Photo upload error."
	Message[PHOTO_NOT_EXIST] 		= "Photo does not exist."
	Message[PHOTO_DELETE_SUCCESS] 	= "Photo delete success."
	Message[PHOTO_GET_SUCCESS]		= "Photo get success."
}

// Translate a response code to a detailed message.
func GetMessage(code int) string {
	msg, ok := Message[code]
	if ok {
		return msg
	}
	return ""
}
```
这里要注意的是，我们不仅定义了返回码，还为每个返回码定义了对应的详细信息。当然，不定义详细信息也不是不可以，但是那样的话对客户端的可读性就很差，每次想查某个返回码对应的是什么信息时都得翻查文档。

## JWT
定义完配置文件&常量后，我们来实现一下`JWT`。`JWT = JSON Web Token`，是用来做鉴权的一种手段。毕竟我们辛辛苦苦写好，部署好的API不是随便一个人都可以任意调用的，只有通过鉴权的调用才是真正的合法调用。关于`JWT`的详细解释，可以查看我之前的一篇文章[《[Golang]JWT及其Golang实现》](https://marcoma.xyz/2019/01/27/jwt/)。

因为`JWT`还不算我们业务逻辑里的模块，将其看作`utils`就行，所以要在项目路径下新建`utils`目录，并新建`jwt.go`。

```
gin-photo-storage
|—— conf
 	|—— server.conf
 	|—— cfg.go
|—— constant
	|—— constant.go
	|—— resp_code.go
|—— utils
	|—— jwt.go
```
至于`JWT`的具体实现，因为它涉及一些数字签名的算法，造轮子手写大概率不是一个好的选择，所以我们直接用一个第三方的`JWT`库——`github.com/dgrijalva/jwt-go`来实现。

+ Golang <= 1.10且项目位于`$GOPATH/src`下的可以直接用go get来安装：`go get -u github.com/dgrijalva/jwt-go`
+ Golang >= 1.11且使用go mod来管理package的可以直接修改`go.mod`然后`go mod init <module-name>`

```golang
// 以下为jwt.go的内容
package utils

import (
	"gin-photo-storage/conf"
	"gin-photo-storage/constant"
	"github.com/dgrijalva/jwt-go"
	"log"
	"time"
)

// self-defined user claim
type UserClaim struct {
	UserName string `json:"user_name"`
	jwt.StandardClaims
}

// Generate a JWT based on the user name.
func GenerateJWT(userName string) (string, error) {
	// define a user claim
	claim := UserClaim{
		userName,
		jwt.StandardClaims{
			Issuer:    constant.PHOTO_STORAGE_ADMIN,
			ExpiresAt: time.Now().Add(constant.JWT_EXP_MINUTE * time.Minute).Unix(),
		},
	}

	// generate the claim and the digital signature
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claim)
	jwtString, err := token.SignedString([]byte(conf.ServerCfg.Get(constant.JWT_SECRET)))
	if err != nil {
		log.Fatalln("JWT generation error")
		return "", err
	}
	return jwtString, nil
}

// Parse a JWT into a user claim.
func ParseJWT(jwtString string) (*UserClaim, error) {
	token, err := jwt.ParseWithClaims(jwtString, &UserClaim{}, func(token *jwt.Token) (interface{}, error) {
		return []byte(conf.ServerCfg.Get(constant.JWT_SECRET)), nil
	})

	if token != nil && err == nil {
		if claim, ok := token.Claims.(*UserClaim); ok && token.Valid {
			return claim, nil
		}
	}
	return nil, err
}
```
实现`JWT`的核心点有3个：

+ 自定义一个claim的结构，在这里就是`UserClaim`，其中简单地包含一个用户名。
+ 实现生成`JWT`的逻辑，也就是传入自定义claim的字段，为其生成一个`JWT`字符串。
+ 实现解析`JWT`的逻辑，也就是传入一个`JWT`字符串，反生成出一个自定义的claim。

## 总结

发散思考：

+ 配置文件`server.conf`放在`conf`文件夹里，往git仓库上提交代码时，应不应该把配置文件也提交上去？如果不应该提交，要怎么设置？

	+ 关键词：git命令、git ignore

+ 跟hard code在代码里相比，配置文件已经灵活了无数倍，但还有没有更好的方案来存储&读取配置？假如想动态更新配置值，不用重启程序就能读取到最新的修改值，要怎么做？

	+ 关键词：配置中心、热更新

+ `JWT`有什么功能上的缺点？有没有别的鉴权机制可以克服`JWT`的缺点？多种鉴权方式互相对比各有什么优劣？

	+ 关键词：`Cookie-Session`、主动过期、单点登录

+ 用`JWT`的安全性如何？常见的web攻击手段能不能攻破`JWT`？

	+ 关键词：`CSRF`、`XSS`

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
