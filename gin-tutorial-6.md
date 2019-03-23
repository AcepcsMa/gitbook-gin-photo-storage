
## Overview
在上一篇教程[《[Gin教程-5]中间件与API开发-2》](https://marcoma.xyz/2019/03/07/gin-tutorial-5/)里我们已经开发完基本的API，并实现了Router和Server，成功把服务端程序跑起来了。这时候我们就要审视一下，是否必要的API真的都全部实现了呢？

其实不然，还记得我们的Photo Model里有`Tag`和`Description`两个文本类型的字段。对于文本数据我们肯定会有搜索（模糊搜索）的需求，例如我们可能会给一张旅游的照片加上某年某月某日在某处做了什么的Description文本，当存储的图片越来越多，我们就要通过搜索Description来找出想要的图片。

面对搜索的需求，其中一种解决方案是利用MySQL作文本搜索，例如`select ... from photo where description like '%xxx%';`，但是这样的查询效率确实不高，而且不能实现模糊搜索。**所谓的不能模糊搜索，就是只能精准匹配，不存在分词、理解、transform的过程。**

例如某张照片的Description是“2019年3月10日，我们团队一起在黄山的空地上玩扑克”，过了很久之后用户忘了该图片的具体Description是怎么写的，只模糊地记得“打扑克”，当他搜索“打扑克”，对应的查询语句就是`select ...from photo where description like '%打扑克%';`，结果什么都没查出来。

基于这样的需求和问题，我们不能用MySQL实现搜索功能，应转而使用Elasticsearch。

**Note：本教程不包含Elasticsearch安装和运行的步骤，请自行安装并确保正常运行。**

## 什么是Elasticsearch
根据[Elastic官方](https://www.elastic.co/products/elasticsearch)的定义：

> Elasticsearch is a distributed, RESTful search and analytics engine capable of solving a growing number of use cases. As the heart of the Elastic Stack, it centrally stores your data so you can discover the expected and uncover the unexpected.

简而言之，Elasticsearch就是一个可以存储数据，并提供搜索和分析接口的搜索引擎（组件）。

既然Elasticsearch里也可以存储数据，我们就来看看它和MySQL在概念上的相似之处：

+ index ≈ table
+ document ≈ record
+ field ≈ column

在概念上，MySQL里的一张表类似于Elasticsearch里的一个index，MySQL表里的一行记录类似于Elasticsearch里的一个document，MySQL record里的一个字段类似于Elasticsearch document里的一个字段。

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-es-structure.png)

如上图所示，基于存储好的数据，我们可以对它们进行搜索。不同于MySQL的“匹配”，Elasticsearch支持的是真正意义上的“搜索”。它会对你的query进行分词、组合、transform，再去匹配数据里的对应字段的值，不是简单的匹配。

接下来的教程里将使用`Elasticsearch 6.6.1`来实现搜索功能。

## 实现ES Utils
+ Golang <= 1.10且项目位于`$GOPATH/src/`下的，可直接`go get -u github.com/elastic/go-elasticsearch`
+ 否则可用go mod来安装 

首先在`gin-photo-storage/conf/server.conf`里加上ES的配置项，`ES_HOST`和`ES_PORT`填入你自己的ES的host和port：

```
{
    ......
    "ES_HOST": "......",
    "ES_PORT": "......",
    "ES_PHOTO_INDEX": "photo"
    ......
}
```
然后在`gin-photo-storage/constant/constant.go`和`gin-photo-storage/constant/resp_code.go`里加上对应的常量和返回码：

常量定义：

```golang
// 在constant.go里补充

// Elasticsearch constants
ES_HOST 		= "ES_HOST"
ES_PORT 		= "ES_PORT"
ES_PHOTO_INDEX 	= "ES_PHOTO_INDEX"
SEARCH_BY_TAG	= "tags"
SEARCH_BY_DESC	= "description"
```
返回码：

```golang
// 在resp_code.go里补充

// ......
PHOTO_SEARCH_BY_TAG_SUCCESS 	= 4009
PHOTO_SEARCH_BY_DESC_SUCCESS	= 4010
// ......

func init() {
    // ......
    Message[PHOTO_SEARCH_BY_TAG_SUCCESS] = "Photo search by tag success."
	Message[PHOTO_SEARCH_BY_DESC_SUCCESS] = "Photo search by description success."
}
```
接下来在`gin-photo-storage/models/`下新建`es.go`代码文件：

```golang
package models

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"gin-photo-storage/conf"
	"gin-photo-storage/constant"
	"github.com/elastic/go-elasticsearch"
	"github.com/elastic/go-elasticsearch/esapi"
	"log"
	"strings"
)

var ESClient *elasticsearch.Client
var PhotoIndexingError = errors.New("photo indexing error")
var PhotoSearchError = errors.New("photo search error")
var PhotoUpdateError = errors.New("photo update error")

// search request body
var SearchRequest = `{
	"query": {
		"bool": {
			"must": [
				{
					"match": {
						"%s": "%s"
					}
				},
				{
					"term": {
						"auth_id": %d
					}
				}
			]
		}
	}
}`

// update request body
var AddPhotoUrlRequest = `{
	"doc": {
		"url": "%s"
	}
}`

// search type which indicates if we are searching by tag or by description
type SearchType string

// Photo struct used in elasticsearch.
type PhotoToIndex struct {
	AuthID		uint		`json:"auth_id"`
	BucketID	uint		`json:"bucket_id"`
	ID 			uint		`json:"id"`
	Name 		string		`json:"name"`
	Tags 		[]string	`json:"tags"`
	Url			string		`json:"url"`
	Description string		`json:"description"`
}

// Init elasticsearch client.
func init() {

	host := conf.ServerCfg.Get(constant.ES_HOST)
	port := conf.ServerCfg.Get(constant.ES_PORT)
	esCfg := elasticsearch.Config{
		Addresses: []string{
			fmt.Sprintf("http://%s:%s", host, port),
		},
	}
	var err error
	ESClient, err = elasticsearch.NewClient(esCfg)
	if err != nil {
		log.Fatalln(err)
	}
}

// Index a photo in elasticsearch.
func IndexPhoto(photo *Photo) error {

	// the document we want to index
	photoToIndex := PhotoToIndex{
		AuthID: photo.AuthID,
		BucketID: photo.BucketID,
		ID: photo.ID,
		Name: photo.Name,
		Tags: strings.Split(photo.Tag, ";"),
		Url: photo.Url,
		Description: photo.Description,
	}
	body, _ := json.Marshal(&photoToIndex)

	// set up index request
	request := esapi.IndexRequest{
		Index: conf.ServerCfg.Get(constant.ES_PHOTO_INDEX),
		DocumentID: fmt.Sprintf("%d", photoToIndex.ID),
		Body: bytes.NewReader(body),
		Refresh: "true",
	}

	if res, err := request.Do(context.Background(), ESClient); err == nil {
		defer res.Body.Close()
		if res.IsError() {
			log.Println("Photo indexing error")
			return PhotoIndexingError
		}
	} else {
		log.Println(err)
		return PhotoIndexingError
	}
	return nil
}

// Add the photo url in elasticsearch.
func AddPhotoUrl(photoID uint, url string) error {
	queryBody := fmt.Sprintf(AddPhotoUrlRequest, url)

	res, err := ESClient.Update(
		conf.ServerCfg.Get(constant.ES_PHOTO_INDEX),
		fmt.Sprintf("%d", photoID),
		strings.NewReader(queryBody),
		)

	if err != nil {
		log.Println(err)
		return PhotoUpdateError
	}

	if res.IsError() {
		log.Println(PhotoUpdateError)
		return PhotoUpdateError
	} else {
		defer res.Body.Close()
		resMap := make(map[string]interface{})
		if err := json.NewDecoder(res.Body).Decode(&resMap); err != nil {
			log.Println(err)
			return PhotoUpdateError
		} else {
			if fmt.Sprintf("%d", resMap["updated"]) != "0" {
				return nil	// update success
			}
		}
	}
	return PhotoUpdateError
}

// Search photo(s) by the given field
// 1. searchType = SEARCH_BY_TAG, the field is a tag
// 2. searchType = SEARCH_BY_DESC, the field is a description
func SearchPhoto(field string, authID uint, offset int, searchType SearchType) ([]PhotoToIndex, error) {
	queryBody := fmt.Sprintf(SearchRequest, searchType, field, authID)
	photos := make([]PhotoToIndex, 0, constant.PAGE_SIZE)

	res, err := ESClient.Search(
		ESClient.Search.WithContext(context.Background()),
		ESClient.Search.WithIndex(conf.ServerCfg.Get(constant.ES_PHOTO_INDEX)),
		ESClient.Search.WithBody(strings.NewReader(queryBody)),
		ESClient.Search.WithFrom(offset),
		ESClient.Search.WithSize(constant.PAGE_SIZE),
		)

	if err != nil {
		log.Println(err)
		return photos, PhotoSearchError
	}

	if res.IsError() {
		log.Println(PhotoSearchError)
		return photos, PhotoSearchError
	} else {
		defer res.Body.Close()
		resMap := make(map[string]interface{})
		if err := json.NewDecoder(res.Body).Decode(&resMap); err != nil {
			log.Println(err)
			return photos, PhotoSearchError
		} else {
			// for each hit in the response, we marshal the source into the photo object
			for _, hit := range resMap["hits"].(map[string]interface{})["hits"].([]interface{}) {
				source, _ := json.Marshal(hit.(map[string]interface{})["_source"])
				photo := PhotoToIndex{}
				json.Unmarshal(source, &photo)
				photos = append(photos, photo)
			}
		}
	}

	return photos, nil
}
```
在`es.go`里，主要实现了以下几个逻辑：

+ 读取配置项，新建ES client
+ index图片
+ 更新图片url
+ 通过tag或者description来搜索图片（定义了SearchType，取值要么是tag要么是desc）

其中涉及到一些go client的API使用（index、search）还有对ES返回结果JSON的解析，对ES go client或者API使用感兴趣的可参考它们的项目地址：[elastic/go-elasticsearch](https://github.com/elastic/go-elasticsearch)。

## 整合实现搜索功能
实现了ES Utils后，可以把ES Utils和之前的代码整合到一起来实现搜索功能。

首先在`AddPhoto()`里加上index图片的代码：

```golang
// 补充models/photo.go
// Add a new photo
func AddPhoto(photoToAdd *Photo, photoFileHeader *multipart.FileHeader) (*Photo, string, error) {
	// ......

	// index a photo in elasticsearch
	if err = IndexPhoto(&photo); err != nil {
		log.Println(err)
		return nil, "", err
	}

	// ......
}
```
然后在`UpdatePhoto()`里加上更新ES的代码：

```golang
// 补充models/photo.go
// Update a photo.
func UpdatePhoto(photoToUpdate *Photo) (*Photo, error) {
	// ......

	// update elasticsearch
	updated := Photo{}
	trx.Where("id = ?", photoToUpdate.ID).First(&updated)
	if err := IndexPhoto(&updated); err != nil {
		return &photo, err
	}

	return &updated, nil
}
```
最后实现一个图片搜索的API：

```golang
// 补充apis/v1/photo.go
// Search a photo (by tag / description)
func SearchPhoto(context *gin.Context) {
	responseCode := constant.INVALID_PARAMS
	authID, err := strconv.Atoi(context.Query("auth_id"))
	tag, tagExisted := context.GetQuery("tag")
	desc, descExisted := context.GetQuery("desc")
	if err != nil || (tagExisted && descExisted) || (!tagExisted && !descExisted) {
		context.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
			"code": responseCode,
			"data": make(map[string]string),
			"msg":  constant.GetMessage(responseCode),
		})
	}

	var searchType models.SearchType
	var field string
	if tagExisted {
		searchType = constant.SEARCH_BY_TAG
		field = tag
	} else {
		searchType = constant.SEARCH_BY_DESC
		field = desc
	}

	validCheck := validation.Validation{}
	validCheck.Min(authID, 1, "auth_id").Message("Auth id must be positive")
	validCheck.MinSize(field, 1, "search_field").Message("Search field can't be empty")

	data := make(map[string]interface{})
	if !validCheck.HasErrors() {
		offset := context.GetInt("offset")
		if photos, err := models.SearchPhoto(field, uint(authID), offset, searchType); err == nil {
			data["photos"] = photos
			responseCode = constant.PHOTO_SEARCH_BY_TAG_SUCCESS
		} else {
			responseCode = constant.INTERNAL_SERVER_ERROR
		}
	} else {
		for _, err := range validCheck.Errors {
			log.Println(err)
		}
	}

	context.JSON(http.StatusOK, gin.H{
		"code": responseCode,
		"data": data,
		"msg": constant.GetMessage(responseCode),
	})
}
```
**Note：一个搜索API可分别支持两种类型的搜索，但不支持同时按tag和description搜索。**

把上述搜索API添加到Router规则中：

```golang
// 补充routers/router.go
// Init router, adding paths to it.
func init() {
	// ......

	// api group for v1
	v1Group := Router.Group("/api/v1")
	{
		// ......

		// api group for photo
		photoGroup := v1Group.Group("/photo")
		{
			// ......
			photoGroup.GET("/search", checkAuthMdw, refreshMdw, paginationMdw, v1.SearchPhoto)
		}
	}
}
```
同之前的API，在执行`SearchPhoto()`之前，要先执行auth、refresh、pagination中间件。启动Server，用Postman访问该API进行搜索测试即可。

## 总结

+ Elasticsearch和MySQL都可以存数据，都可以支持复杂查询，它们的核心差别有哪些？

    + 关键词：搜索引擎、数据结构、事务、Lucene

+ 在教程里，我们提到要么手写DSL查询ES，要么利用各编程语言的ES client来查询ES，那可不可以用SQL或者类SQL语句来查询ES呢？如果要你自己实现，可以怎么实现？

    + 关键词：Elasticsearch SQL support、SQL parser

+ MySQL InnoDB里支持全文索引，使用全文索引可不可以支持我们的搜索需求？

    + 关键词：全文索引、InnoDB

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
