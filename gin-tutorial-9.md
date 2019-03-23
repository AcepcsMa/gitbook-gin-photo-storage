
## Overview

截至上一篇教程 [《[Gin教程-8]接入swagger》](https://marcoma.xyz/2019/03/19/gin-tutorial-8/)，从代码的层面而言，整个项目已经完成了。但是写完了代码，写完了测试用例，通过了测试，即使连文档也全部完成了也不代表整个项目的结束。

**完成了设计、编码、文档编写后，接下来自然就是项目的部署。**

在整个开发过程中，服务端通信我们默认是走http协议。但随着https的流行，且https本身在安全性上有优势，新项目应优先使用https协议。另外，在前面的教程中，我们写完API代码，都是直接运行服务端程序，用`Postman`等工具直接访问服务端程序进行测试。这样“裸连”API，在单机部署或者访问量较小的情况下没什么问题，但当访问量剧增，单机部署撑不住而必须要多机部署时，“裸连”API就不太合适了。

所以，本篇以及接下来的最后一篇教程将带大家思考和讨论部署方面的问题。

## nginx

上面提到，当访问量很大从而需要多机部署时，客户端是不知道应该连哪一台后端服务器的（不可能提前把所有服务器地址hard code到客户端中）。所以“裸连”API是不可行的，如下图：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-nginx.png)

所以，我们要在后端服务器和客户端之间多加一层组件，当后端服务器多于1台时，该组件可以把用户请求分发到多台后端服务器中的一台。甚至该组件还可以监测每一台后端服务器的访问压力，每次尽可能把请求分发到压力最小的服务器上（所谓的负载均衡）。如下图：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-nginx-2.png)

只要中间的分发组件的地址是固定的，客户端只需要知道分发组件的地址，把请求发给该组件，该组件会自动地把请求分发给某台后端服务器，调用API，继而原路返回给客户端。

**这也就是为什么我们要使用nginx。**（关于nginx的入门可参考 [《NGINX Beginner's Guide》](http://nginx.org/en/docs/beginners_guide.html)）

**Note：本教程不涉及nginx的安装，请自行查阅安装教程，并确保nginx在开发主机上成功安装。**

## https
接下来看看如何把服务端协议从http改成https。

https主打安全，核心是加密（其实是非对称加密）。既然要实现非对称加密，那肯定得引入**公钥、私钥、证书**等概念。所以先来看看怎么生成我们所需的秘钥和证书。

（关于https的相关知识，本人推荐一个通俗易懂的漫画讲解：[《漫画：什么是 HTTPS 协议？》](https://juejin.im/post/5c889918e51d45346459994d)）

**Note：执行以下命令前，请先确保成功安装了`openssl`。**

---

第一步，任意位置新建目录`https`，在该目录下创建root key：

```
openssl genrsa -des3 -out rootCA.key 2048
```
**Note：请记住生成root key时输入的密码，下面几步需要输入！**

第二步，基于root key生成根SSL证书，有效期设为一年（365天）：

```
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 365 -out rootCA.pem
```
过程中会提示输入以下信息（可根据自己的实际信息填写）：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-https-ssl.png)

第三步，在`https`目录下新建一个`v3.ext`文件：

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName=@alt_names

[alt_names]
DNS.1 = localhost
```
第四步，在`https`目录下创建密钥：

```
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key
```
过程中会提示输入以下信息（可根据自己的实际信息填写）：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-https-ssl-2.png)

第五步，在`https`目录下生成证书文件，有效期设为一年（365天）：

```
openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 365 -sha256 -extfile v3.ext
```
发现在`https`目录下已成功生成`server.key`和`server.crt`。

## 整合测试
**Note：请先确保`MySQL`、`Redis`、`Elasticsearch`等组件已正常运行中。**

---

第一步，把之前生成的`server.key`和`server.crt`复制到`gin-photo-storage/conf`目录下，然后修改服务端程序`main.go`的代码：

```golang
// ......
// 注释掉server.ListenAndServe()
server.ListenAndServeTLS("conf/server.crt", "conf/server.key")
```
第二步，启动服务端程序。

第三步，停止nginx服务，修改nginx配置文件。先看看当前使用的是哪个配置文件：

```
nginx -t
```
该命令会打印出nginx所使用的配置文件路径。

第四步，修改该nginx配置文件：

```
worker_processes 1;

events {
    worker_connections  1024;
}

http {
    # ...... 保留原默认配置即可
    
    # 配置upstream以实现负载均衡
    # 当部署了多台后端服务器时，这里可填写多个地址，实现负载均衡
    upstream api_service {
        server localhost:9088; # 服务端监听了9088端口，所以这里用9088
    }
    
    # 配置server
    server {
        listen       443 ssl;   # nginx监听443端口
        server_name  localhost;

        ssl_certificate      /....../server.crt; # 填写server.crt的绝对路径
        ssl_certificate_key  /....../server.key; # 填写server.key的绝对路径

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
			proxy_pass https://api_service/; # 转发到api_service处，也就是上面定义的upstream
        }
    }
}
```
**Note：我的配置文件`server.conf`里用的是`9088`端口，读者可根据自己的设置修改。**

**Note：如果我们的后端服务器只有一台，自然就不存在什么“负载均衡”了，这种情况下可以不配置upstream，而是直接把后端服务的地址填到`proxy_pass`中：**

```
http {
    # ......
    server {
        listen 443 ssl;
        server_name localhost;
        
        # ......
        
        location / {
            proxy_pass https://localhost:9088/;
        }
    }
}
```
第五步，保存配置文件后，启动nginx。

第六步，用`Postman`工具测试：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-nginx-test.png)

可以看到，我们访问的是443端口的`/api/v1/auth/check`，而不再是服务端监听的9088端口，成功得到API调用后的返回结果。

## 总结

+ nginx除了用作反向代理（把客户端请求反向转发给服务端），还能用来做什么？

    + 关键词：静态服务器、拦截器、缓存、Lua

+ 当我们在nginx配置里的upstream里设置了多个后端服务的地址，相当于让nginx实现负载均衡，让它把用户请求尽量均衡地分发给各台服务器。那么nginx的负载均衡有哪些模式呢？

    + 关键词：nginx负载均衡模式、负载均衡算法

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
