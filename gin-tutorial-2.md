
## Overview

在上一篇教程[《[Gin教程-1]photo gallery项目》](https://marcoma.xyz/2019/02/23/gin-tutorial-1/)里，我们已经完成了：

+ 配置文件 & 配置解析
+ 常量定义
+ `JWT`的实现

本篇将要开始真正的“业务开发”，讲讲数据库表的定义和对应model的实现。

## 组件&技术
数据库我们用的是`MySQL 5.7`，ORM框架用的是`gorm`。

为什么我们用`MySQL`呢？其实真正要用的是“关系型数据库”，因为我们的业务非常适合用关系型数据库来实现。粗略想一想，我们做的是图片存储库，大概要存这么些东西：

+ 用户信息
+ 图片所属的bucket数据
+ 图片本身的数据
+ ......

不难想到，在业务逻辑层面，**一个用户可以创建多个bucket，一个bucket里可以存多张图片，这就形成了“关系”，所以用关系型数据库是没错的。**不过，本项目用的是`MySQL`，用别的关系型数据库如`MariaDB`、`PostgreSQL`、`Microsoft Sql Server`也并无太大差别。（它们本身的差别还是不小的，但是在本项目的业务层面没什么差别）

至于为什么要用`gorm`呢？其实Golang的ORM框架还是有不少的，具体ORM框架之间的对比文章也有很多，本项目选择`gorm`一是因为`EDDYCJY`的教程中是用的`gorm`，二是自己一路用下来确实觉得还不错，所以就继续沿用`gorm`了。如果读者有别的用习惯了的ORM框架，例如`beego orm`或者`xorm`，也可以自行替换，应该不会出什么问题。

## 数据库
1. E-R图定义

	E-R图，就是实体关系图。为什么要在定义数据库表之前先画好E-R图呢？一是为了帮自己理清楚各个实体之间的关系，二是为了规范开发流程，以后有据可循。如果是团队合作，定义数据库表之前不画好E-R图，之后在开发过程中很可能出一大堆幺蛾子。
	
	基于我们对“图片存储库”的业务描述，系统里的实体目前就是3个：
	
	+ 用户，一个用户可以有多个bucket
	+ bucket，一个bucket可以存多张图片
	+ 图片，一张图片只能属于一个用户

	对应的E-R图如下：
	
	![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-er-diagram.png)
	
2. 数据库表定义

	```sql
	create database photo_gallery;
	
	use photo_gallery;
	
	# 用户表
	drop table if exists auth;
	create table auth
	(
		id int primary key auto_increment,
		user_name varchar(16) unique not null,
		password varchar(255) not null,
		email varchar(128) not null,
		created_at timestamp default CURRENT_TIMESTAMP,
		updated_at timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
	);
	
	# bucket表
	drop table if exists bucket;
	create table bucket
	(
		id int primary key auto_increment,
		auth_id int,
		name varchar(64) not null,
		state tinyint(1) default 1,
		size int default 0,
		description text,
		created_at timestamp default CURRENT_TIMESTAMP,
		updated_at timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
		CONSTRAINT UC_bucket UNIQUE(auth_id, name),
		INDEX idx_aid_name (auth_id, name)
	);
	
	# 图片表
	drop table if exists photo;
	create table photo
	(
		id int primary key auto_increment,
		bucket_id int,
		auth_id int,
		name varchar(255) not null,
		tag varchar(255),
		url varchar(255) not null,
		description text,
		state tinyint(1) default 1,
		created_at timestamp default CURRENT_TIMESTAMP,
		updated_at timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
		constraint UC_photo UNIQUE(bucket_id, name),
		INDEX idx_bid_name (bucket_id, name)
	);
	```
	除了E-R图里标注的属性外，每个实体对应的表里都有共同的属性：`created_at`和`updated_at`，这是为了debug的时候更加方便地看到数据修改时间，一般都建议带上这两个字段。
	
3. 实现数据库连接

	定义完了数据库表，我们可以用Golang实现一个数据库连接，开始和数据库进行交互。先在项目路径下新建`models`文件夹，然后在`models`下新建`db.go`文件。
	
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
	|—— models
		|—— db.go 
	```
	在`db.go`里，我们要
	
	+ 读取数据库配置，如`Host`、`User`、`Password`等......
	+ 根据配置建立起一个全局的连接，可供其他模块使用

	```golang
	package models
	
	import (
		"fmt"
		"gin-photo-storage/conf"
		"gin-photo-storage/constant"
		"gin-photo-storage/utils"
		_ "github.com/go-sql-driver/mysql"	// remember to import mysql driver
		"github.com/jinzhu/gorm"
		"log"
		"strconv"
		"strings"
		"time"
	)
	
	var db *gorm.DB
	
	// Init the database connection.
	func init() {
		dbType := conf.ServerCfg.Get(constant.DB_TYPE)
		dbHost := conf.ServerCfg.Get(constant.DB_HOST)
		dbPort := conf.ServerCfg.Get(constant.DB_PORT)
		dbUser := conf.ServerCfg.Get(constant.DB_USER)
		dbPwd := conf.ServerCfg.Get(constant.DB_PWD)
		dbName := conf.ServerCfg.Get(constant.DB_NAME)
	
		var err error
		db, err = gorm.Open(dbType, fmt.Sprintf(constant.DB_CONNECT, dbUser, dbPwd, dbHost, dbPort, dbName))
		if err != nil {
			log.Fatalln("Fail to connect database!")
		}
	
		db.SingularTable(true)
		if !db.HasTable(&Auth{}) {
			db.CreateTable(&Auth{})
		}
		if !db.HasTable(&Bucket{}) {
			db.CreateTable(&Bucket{})
		}
		if !db.HasTable(&Photo{}) {
			db.CreateTable(&Photo{})
		}
	}
	```
	**⚠️Note：记得要先安装mysql的Golang驱动：`github.com/go-sql-driver/mysql`**。
	
## Model定义
有了数据库表之后，每张表可以对应来实现一个Model，即`Auth`、`Bucket`、`Photo`。由于多个Model都有一些共同的属性可以抽象出来，如`id`、`created_at`、`updated_at`这种大家都有的，就没必要重复在每个结构体内都定义一遍。所以我们要定义一个包含共同属性的Model：`BaseModel`。

先在`models`下新建`auth.go`、`bucket.go`、`photo.go`三个代码文件。

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
|—— models
	|—— auth.go
	|—— bucket.go
	|—— db.go 
	|—— photo.go
```
然后先在`db.go`里实现上面提到的`BaseModel`，之后再逐个实现`Auth`、`Bucket`、`Photo`。

1. BaseModel

	```golang
	// 在db.go里实现
	// The base model of all models, including ID & CreatedAt & UpdatedAt.
	type BaseModel struct {
		ID 			uint 		`json:"id" gorm:"primary_key;AUTO_INCREMENT" form:"id"`
		CreatedAt 	time.Time 	`json:"created_at" gorm:"default: CURRENT_TIMESTAMP" form:"created_at"`
		UpdatedAt 	time.Time 	`json:"updated_at" gorm:"default: CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP" form:"updated_at"`
	}
	```
	其实，`gorm`的开发者也想到了这种抽象出共同属性的Model，所以框架里早已定义好了`gorm.Model`结构体，包含了`ID`、`CreatedAt`、`UpdatedAt`、`DeletedAt`。但是在这里我们不需要`DeletedAt`字段，所以就没有直接用它提供的Model，而是自行定义。
2. Auth

	```golang
	// 在auth.go里实现
	package models
	
	import (
		"crypto/md5"
		"fmt"
		"github.com/pkg/errors"
		"io"
	)
	
	type Auth struct {
		BaseModel
		UserName 	string `json:"user_name" gorm:"type:varchar(16)"`
		Password 	string `json:"password" gorm:"type:varchar(16)"`
		Email 		string `json:"email" gorm:"type:varchar(128)"`
	}
	
	var AuthExistsError = errors.New("auth already exists")
	
	// Add a new auth.
	func AddAuth(username, password, email string) error {
		trx := db.Begin()
		defer trx.Commit()
	
		auth := Auth{}
		trx.Set("gorm:query_option", "FOR UPDATE").
			Where("user_name = ?", username).
			First(&auth)
		if auth.ID > 0 {
			return AuthExistsError
		}
	
		hash := md5.New()
		io.WriteString(hash, password)	// for safety, don't just save the plain text
		auth.UserName = username
		auth.Password = fmt.Sprintf("%x", hash.Sum(nil))
		auth.Email = email
		err := trx.Create(&auth).Error
		if err != nil {
			return err
		}
		return nil
	}
	
	// Check if the auth is valid.
	func CheckAuth(username, password string) bool {
		trx := db.Begin()
		defer trx.Commit()
	
		hash := md5.New()
		io.WriteString(hash, password)
		password = fmt.Sprintf("%x", hash.Sum(nil))	//	for safety, don't just save the plain text
		auth := Auth{}
		trx.Set("gorm:query_option", "FOR UPDATE").
			Where("user_name = ? AND password = ?", username, password).
			First(&auth)
		if auth.ID > 0 {
			return true
		}
		return false
	}
	```
	Auth里需要注意的是，注册时**不能存明文密码**，登陆校验时也不能直接用明文校验，需要用`md5`加一层壳，这在一定程度上能防止后端数据库被一锅端导致的密码泄露。
3. Bucket

	```golang
	// 在bucket.go里实现
	package models
	
	import (
		"gin-photo-storage/constant"
		"github.com/pkg/errors"
		"log"
	)
	
	// The bucket model.
	type Bucket struct {
		BaseModel
		AuthID 		uint	`json:"auth_id" gorm:"type:int" form:"auth_id"`
		Name 		string	`json:"bucket_name" gorm:"type:varchar(64)" form:"bucket_name"`
		State 		int		`json:"state" gorm:"type:tinyint(1)" form:"state"`
		Size 		int		`json:"size" gorm:"type:int" form:"bucket_size"`
		Description string	`json:"description" gorm:"type:text" form:"description"`
	}
	
	var BucketExistsError = errors.New("bucket already exists")
	var NoSuchBucketError = errors.New("no such bucket")
	
	// Add a new bucket.
	func AddBucket(bucketToAdd *Bucket) error {
		trx := db.Begin()
		defer trx.Commit()
	
		// check if the bucket exists, select with a WRITE LOCK.
		bucket := Bucket{}
		trx.Set("gorm:query_option", "FOR UPDATE").
			Where("auth_id = ? AND name = ? AND state = ?", bucketToAdd.AuthID, bucketToAdd.Name, 1).
			First(&bucket)
		if bucket.ID > 0 {
			return BucketExistsError
		}
	
		bucket.AuthID = bucketToAdd.AuthID
		bucket.Name = bucketToAdd.Name
		bucket.State = 1
		bucket.Size = 0
		bucket.Description = bucketToAdd.Description
		if err := trx.Create(&bucket).Error; err != nil {
			log.Println(err)
			return err
		}
		return nil
	}
	
	// Delete an existed bucket.
	func DeleteBucket(bucketID uint) error {
		trx := db.Begin()
		defer trx.Commit()
	
		result := trx.Where("id = ? and state = ?", bucketID, 1).Delete(Bucket{})
		if err := result.Error; err != nil {
			return err
		}
		if affected := result.RowsAffected; affected == 0 {
			return NoSuchBucketError
		}
		return nil
	}
	
	// Update an existed bucket.
	func UpdateBucket(bucketToUpdate *Bucket) error {
		trx := db.Begin()
		defer trx.Commit()
	
		bucket := Bucket{}
		bucket.ID = bucketToUpdate.ID
		result := trx.Model(&bucket).Updates(*bucketToUpdate)
		if err := result.Error; err != nil {
			return err
		}
		if affected := result.RowsAffected; affected == 0 {
			return NoSuchBucketError
		}
		return nil
	}
	
	// Get a bucket by bucket id.
	func GetBucketByID(bucketID uint) (Bucket, error) {
		trx := db.Begin()
		defer trx.Commit()
	
		bucket := Bucket{}
		found := NoSuchBucketError
		trx.Where("id = ?", bucketID).First(&bucket)
		if bucket.ID > 0 {
			found = nil
		}
		return bucket, found
	}
	
	// Get all buckets of the given user.
	func GetBucketByAuthID(authID uint, offset int) ([]Bucket, error) {
		trx := db.Begin()
		defer trx.Commit()
	
		buckets := make([]Bucket, 0, constant.PAGE_SIZE)
		err := trx.Where("auth_id = ?", authID).
			Offset(offset).
			Limit(constant.PAGE_SIZE).
			Find(&buckets).Error
	
		if err != nil {
			log.Println(err)
			return buckets, err
		}
		return buckets, nil
	}
	```
	同样地，定义了Bucket结构体外，还定义了对它增/删/改/查的接口。
3. Photo

	```golang
	// 在photo.go里实现
	package models
	
	import (
		"bufio"
		"gin-photo-storage/constant"
		"gin-photo-storage/utils"
		"github.com/jinzhu/gorm"
		"github.com/pkg/errors"
		"log"
		"mime/multipart"
	)
	
	var NoSuchPhotoError = errors.New("no such photo")
	var PhotoExistsError = errors.New("photo already exists")
	var PhotoFileBrokenError = errors.New("photo file is broken")
	
	// The photo model.
	type Photo struct {
		BaseModel
		AuthID 		uint		`json:"auth_id" gorm:"type:int" form:"auth_id"`
		BucketID 	uint		`json:"bucket_id" gorm:"type:int" form:"bucket_id"`
		Name 		string		`json:"name" gorm:"type:varchar(255)" form:"name"`
		Tag 		string		`json:"tag" gorm:"type:varchar(255)" form:"tag"`
		Tags 		[]string	`json:"tags" gorm:"-" form:"tags"`
		Url 		string		`json:"url" gorm:"type:varchar(255)" form:"url"`
		Description string		`json:"description" gorm:"type:text" form:"description"`
		State 		int 		`json:"state" gorm:"type:tinyint(1)" form:"state"`
	}
	
	// Add a new photo
	func AddPhoto(photoToAdd *Photo, photoFileHeader *multipart.FileHeader) (*Photo, string, error) {
		trx := db.Begin()
		defer trx.Commit()
	
		// check if the photo exists, select with a WRITE LOCK
		photo := Photo{}
		trx.Set("gorm:query_option", "FOR UPDATE").
			Where("bucket_id = ? AND name = ?", photoToAdd.BucketID, photoToAdd.Name).
			First(&photo)
		if photo.ID > 0 {
			return nil, "", PhotoExistsError
		}
	
		photo.AuthID = photoToAdd.AuthID
		photo.BucketID = photoToAdd.BucketID
		photo.Name = photoToAdd.Name
		photo.Tag = photoToAdd.Tag
		photo.Description = photoToAdd.Description
		photo.State = 1
	
		err := trx.Create(&photo).Error
		if err != nil {
			log.Println(err)
			return nil, "", err
		}
	
		err = trx.Model(&Bucket{}).Where("id = ?", photoToAdd.BucketID).
			Update("size", gorm.Expr("size + ?", 1)).
			Error
		if err != nil {
			trx.Rollback()
			log.Println(err)
			return nil, "", err
		}
	
		// TODO: upload to the tencent cloud COS
		// ......
	}
	
	// Delete a photo by photo id.
	func DeletePhotoByID(photoID uint) error {
		trx := db.Begin()
		defer trx.Commit()
	
		result := trx.Where("id = ? AND state = ?", photoID, 1).Delete(Photo{})
		if err := result.Error; err != nil {
			log.Println(err)
			return err
		}
		if affected := result.RowsAffected; affected == 0 {
			return NoSuchPhotoError
		}
		return nil
	}
	
	// Delete a photo by its bucket id & its name.
	func DeletePhotoByBucketAndName(bucketID uint, name string) error {
		trx := db.Begin()
		defer trx.Commit()
	
		result := trx.Where("bucket_id = ? AND name = ?", bucketID, name).Delete(Photo{})
		if err := result.Error; err != nil {
			return err
		}
		if affected := result.RowsAffected; affected == 0 {
			return NoSuchPhotoError
		}
		return nil
	}
	
	// Update a photo.
    func UpdatePhoto(photoToUpdate *Photo) (*Photo, error) {
    	trx := db.Begin()
    	defer trx.Commit()
    
    	photo := Photo{}
    	photo.ID = photoToUpdate.ID
    
    	result := trx.Model(&photo).Updates(*photoToUpdate)
    	if err := result.Error; err != nil {
    		log.Println(err)
    		return &photo, err
    	}
    	if affected := result.RowsAffected; affected == 0 {
    		return &photo, NoSuchPhotoError
    	}

    	return &photo, nil
    }
	
	// Update the url for a photo.
	func UpdatePhotoUrl(photoID uint, url string) error {
		trx := db.Begin()
		defer trx.Commit()
	
		photo := Photo{}
        photo.ID = photoID
		err := trx.Model(&photo).Update("url", url).Error
		if err != nil {
			return err
		}
		return nil
	}
	
	// Get a photo by its photo id.
	func GetPhotoByID(photoID uint) (*Photo, error) {
		trx := db.Begin()
		defer trx.Commit()
	
		photo := Photo{}
		err := trx.Where("id = ?", photoID).First(&photo).Error
		found := NoSuchPhotoError
		if err != nil || photo.ID == 0 {
			log.Println(err)
			found = err
		}
		found = nil
		return &photo, found
	}
	
	// Get photos by bucket id.
	func GetPhotoByBucketID(bucketID uint, offset int) ([]Photo, error) {
		trx := db.Begin()
		defer trx.Commit()
	
		photos := make([]Photo, 0, constant.PAGE_SIZE)
		err := trx.Where("bucket_id = ?", bucketID).
			Offset(offset).
			Limit(constant.PAGE_SIZE).
			Find(&photos).
			Error
		if err != nil {
			return photos, err
		}
		return photos, nil
	}
	```
	这里要提出三个问题：
	
	1. **⚠️`AddPhoto`里的逻辑并未完善，现在只是往数据库加了一行图片记录，但是还未把图片真正地存下来（上传至云存储）。在下一篇会重点实现这个需求。**
	2. 读者们应该也能发现，在每个Model结构体的每个字段后面，都会跟着一串“标签”（struct tag），标签里面有`json`，有`gorm`，有`form`。简单来说，标签就是用来标注字段的“别名”或者标注字段属性的。`json`标签就是在JSON序列化/反序列化过程中用的，`gorm`标签就是`gorm`框架使用的，`form`标签也是`gorm`框架在绑定请求参数的时候用的。通常，字段的标签会在反射中大量被用到。
	3. 在大项目的开发中，增/删/改/查的接口远远不止这么点，根据业务需求还可以有很多针对特定条件的增/删/改/查接口。具体的接口实现要和你的业务上下游做好对接，例如搞清楚前端需要什么样的数据，或者搞清楚在整个pipeline中你的业务下游需要什么样的数据。

## 总结
思考点：

+ 在多个Add和Update的接口中，入参都是指针类型，而不是常见的值类型，为什么？

	+ 关键词：Golang指针、值复制、节省内存

+ Golang的结构体标签（struct tag）怎么用？

	+ 关键词：struct tag、反射

+ 在`bucket`表里定义了size属性，意味着每次添加图片都要更新bucket size字段。假如把场景扩大，不局限于bucket size，而是一个什么size，在高并发下频繁更新MySQL表里的size字段，性能好吗？如果用缓存存一下，然后再异步落盘到MySQL可以吗？异步落盘可能带来什么问题？

	+ 关键词：缓存、高并发、缓存一致性

+ 用ORM框架有什么好处？ORM框架可能带来什么问题？
+ 数据库表为什么要加索引？索引的原理是什么？

	+ 关键词：InnoDB索引、B+树

+ 关系型数据库是不是万能？在什么场景下不好用？

	+ 关键词：全文检索、模糊搜索、OLAP

+ 为什么把数据库表对应的结构体叫作`Model`？

	+ 关键词：MVC模式

+ 为什么在一些sql查询里要加上`FOR UPDATE`？

	+ 关键词：InnoDB锁机制、并发安全、读写锁

+ 为什么在所有的查询里，都启动一个事务（`trx := db.Begin()`）来执行操作？不用事务，直接查询有什么问题吗？

	+ 关键词：数据库事务，ACID

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
