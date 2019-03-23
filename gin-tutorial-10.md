
## Overview

> **Note：本教程假定读者已有简单的Docker基础，并在本机开发环境中成功安装、运行了Docker Engine。Docker基础知识可参考[《Docker Gitbook》](https://yeasy.gitbooks.io/docker_practice/content/introduction/what.html)。**

在上一篇教程[《[Gin教程-9]》](https://marcoma.xyz/)里，我们已经在本地成功生成公钥私钥和https证书，也成功地部署了nginx，用nginx作反向代理供外部调用我们的API。

但是上篇教程里的部署完全是针对本机环境的，假如我们在本地开发完，需要把整套应用移到Production或者Test环境，那岂不是得到新环境重新把`MySQL`、`Redis`、`Elasticsearch`、`nginx`装一遍，再把项目代码重新编译，手动运行？实在是太麻烦了。

另外，我们把这么多组件全都运行在一台主机上，如果这台主机发生了异常，岂不是所有服务全都挂了？有没有办法能让多个组件分开来提高隔离性，又易于部署呢？

**所以，针对种种不便的情况，我们可以使用Docker来帮助我们解决问题。**



## Docker

1. Docker是什么？

    根据[《Docker Gitbook》](https://yeasy.gitbooks.io/docker_practice/content/introduction/what.html)，Docker就是一种虚拟化技术。以前我们说虚拟化，往往会说在宿主机上开一台虚拟机，在虚拟机里跑一个操作系统（例如Windows上跑一个Ubuntu的虚拟机等等）。但是虚拟机对资源的占用过重，所以后来基于LXC、libcontainer、runC和containerd就开发了Docker——**一种看起来很像虚拟机，但是比虚拟机轻量很多的虚拟化技术。**

2. 为什么要用Docker？

    + 多环境部署方便

        依托于Docker里image和container的概念，只要制作好了image，理论上而言在所有装了Docker Engine的系统里都可以运行该image，然后成功跑起一个很像独立虚拟机的container。一份image，多处运行，部署不同环境只需要加一些环境变量即可，非常方便。
    + 隔离性好

        多个container之间基本相互独立，每个container内部都像是拥有一套独立的系统资源——独立的文件系统、独立的网络栈、独立的块设备、独立的进程表......默认情况下隔离性已经很高了，给人感觉像是运行了多个虚拟机。
    + 资源占用相对较低

        多个container并不需要运行在多个guest OS之上，而是由Docker Engine统一调度和管理。所以即使同时运行几十个container，也不存在几十个guest OS，对系统资源的占用相比虚拟机来说低得多。

3. image和container是什么？

    + image = 镜像，是一种**静态概念**，可类比成一个个写好的代码文件，里面包含了各种静态数据、bin、lib等。
    + container = 容器，是一种**动态概念**，可类比成一个正在run的程序。

    通过`docker run --name <container_name> <image_name>`命令可以基于静态的image构建出一个container，然后让该container运行起来。
4. 怎么制作image？

    一般来说，制作image有两种方法：
    
    + 基于Dockerfile。自己写Dockerfile，然后`docker build ......`构建出一个image。类似于自己写一堆Golang代码，然后`go build`编译出一个可执行程序。Dockerfile的编写规范可参考 [DaoCloud的DCS文档](http://guide.daocloud.io/dcs/dockerfile-9153584.html)。
    + 基于正在运行的container。例如在某个正在运行的container里新建了几个关键文件，然后`docker commit ......`，基于该container构建出一个image。
5. container之间怎么交互（通信）？

    上面提到不同的container之间隔离性很高，即使在一台主机上运行，但看起来就像是多台独立的虚拟机。**可以想象多台机子之间通信，肯定是要通过网络的**，所以常见的通信方式有：
    
    + 让多个容器加入同一个namespace，不同容器使用同一套网络设备和协议。（**准确来说是网络相关的namespace，关于namespace的内容请自行查阅Linux内核相关知识，此处不展开**）
    + 让一台主机上运行的多个容器使用主机的网络namespace，在主机的网络里通信。
    + 多个容器依旧独立，似乎每个容器都拥有着自己的独立网卡，从软件层面设立一个虚拟的网桥（bridge），让多个容器通过该网桥交互数据。

6. container被关闭之后，数据还在吗？

    container被关闭之后，里面的数据是否持久化，**一般取决于启动时有没有挂载（映射）外部目录。**如果在run container的时候，指定了container里某个目录`/xxx/yyy/zzz`映射成某个外部目录`/aaa/bbb/ccc`的话，当container被关闭后，外部目录里会存有所有数据。
    
7. 常用的Docker Client命令

    + `docker pull [<repo_name>]/<image_name>:[<tag_name>]`，从远程仓库拉一个image到本地，默认仓库是Docker Hub，国内访问速度可能会比较慢。
    + `docker run [-Options] <image_name>:[<tag_name>]`，把一个image运行起来成为container，可选加非常多的参数。
    + `docker images`，用来查看Docker Engine中管理着的所有images。
    + `docker ps [-Options]`，用来查看Docker Engine中管理着的所有container。
    + `docker kill <container_name/container_id>`，用来关闭某个正在运行的container，该container变成stop状态。
    + `docker rm <container_name>`，直接删除某个container。
    + `docker rmi <image_name>`，直接删除某个image。
    + `docker exec [-Options] <container_name> <command>`，在某个正在运行的container里执行一条command。

    当然除了上面这些，还有很多Docker Client的命令，这里就不一一展开，毕竟不是专门介绍Docker的教程。详情可参考[《Docker Client官方文档》](https://docs.docker.com/engine/reference/commandline/cli/)。

## 部署MySQL
数据库是整个web项目的核心，所以我们先来用Docker部署`MySQL`。

根据 [MySQL官方镜像](https://hub.docker.com/_/mysql?tab=description) 里给的教程，直接一把梭，只用一条语句就可以把`MySQL`运行起来了：

```shell
docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
```
对应我们的`MySQL`版本（5.7），命令应该是：

```shell
docker run --name mysql -e MYSQL_ROOT_PASSWORD=自定义的root密码 -d mysql:5.7
```
**Note：本教程在数据库中默认使用root用户。**

**然而事情并没有这么简单，除了简单地把`MySQL`的container运行起来，我们还要实现：**

1. **在container启动时，假如数据库和表都不存在，先初始化创建我们之后要用的数据库和表。**
2. **将container内的`MySQL`数据目录映射到一个外部目录，以实现数据持久化，确保不管启停多少次数据依然存在。**

第一步，在任意位置新建目录`image-mysql`，在目录下新建`db.sql`文件，把create database和create table的命令写到`db.sql`里：

```sql
// db.sql
create database if not exists `photo_gallery`;

use `photo_gallery`;

create table if not exists `auth`
(
	id int primary key auto_increment,
	user_name varchar(16) unique not null,
	password varchar(255) not null,
	email varchar(128) not null,
	created_at timestamp default CURRENT_TIMESTAMP,
	updated_at timestamp default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) CHARSET=utf8mb4;

create table if not exists `bucket`
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
) CHARSET=utf8mb4;

create table if not exists `photo`
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
) CHARSET=utf8mb4;
```
第二步，在`image-mysql`目录下新建`Dockerfile`，继承自官方的`MySQL:5.7`镜像，制作自定义镜像。把上面的`db.sql`放到镜像里的`MySQL`的初始化目录下：

```
# Dockerfile
FROM mysql:5.7

COPY db.sql /docker-entrypoint-initdb.d/
```
第三步，在`image-mysql`目录下，基于上面的`Dockerfile`，构建出自定义镜像。镜像名为`my-mysql`，版本标签为`v1.0`：

```shell
docker build -t my-mysql:v1.0 .
```
**Note：命令最后有个`.`，表示基于当前目录下的所有文件（`Dockerfile`、`db.sql`）。**

第四步，用`docker images`查看镜像列表，看看是否制作成功。

第五步，基于`my-mysql:v1.0`镜像启动container，为container取名为mysql，并指定映射某个外部数据目录：

```
docker run \
    --name mysql \
    -e MYSQL_ROOT_PASSWORD=自定义密码 \
    -d \
    -v 外部数据目录:/var/lib/mysql \
    my-mysql:v1.0
```
> 参数说明：
> 
> --name xxx：给container起名叫xxx，可以是任意自定义的名字
> 
> -e xxx=yyy：给container设定环境变量xxx，值为yyy
> 
> -d：后台运行，不要卡死在前台显示
> 
> -v xxx:yyy：把container里的yyy目录映射到本机的xxx目录

第六步，执行`docker ps`看看container是否成功运行。

## 部署Redis
部署好了`MySQL`，对于`Redis`而言其实是非常类似的。由于不需要在`Redis`里初始化什么数据库和表，所以我们无需自制Dockerfile，直接用官方现成的镜像即可，本教程使用`4.0-alpine`版本。

类似地，我们执行：

```
docker run --name redis -d -v 外部数据目录:/data redis:4.0-alpine
```
**Note：`/data`就是该container内部的数据目录。不同的应用，不同的container对应的数据目录就不一样，启动前要先搞清楚。**

另外，使用alpine版本是因为该镜像体积小，去除了很多我们不需要的工具和库，方便使用。读者也可自由选择合适的镜像版本。

## 部署Elasticsearch
接下来可以部署`Elasticsearch`了，由于写本教程时Elasticsearch 6.6.1还未有alpine版本，所以直接使用普通的`6.6.1`版本，Elasticsearch的端口使用默认端口：`9200`。

直接执行：

```
docker run \
    --name es \
    -d \
    -v 外部数据目录:/usr/share/elasticsearch/data \
    elasticsearch:6.6.1
```
同理，也是将其数据目录映射到外部目录，以实现持久化。

## 部署gin项目
因为之前一直是在本地开发，所以很多配置项都直接hard code到配置文件里，真正部署的时候不能这么做。所以先把本项目的配置文件`gin-photo-storage/conf/server.conf`里hard code的配置项改掉，**改成变量的形式**。

第一步，修改`server.conf`：

```
{
    "SERVER_PORT": "<SERVER_PORT>",
    ......
    "DB_HOST": "<DB_HOST>",
    ......
    "REDIS_HOST": "<REDIS_HOST>",
    ......
    "ES_HOST": "<ES_HOST>",
}
```
第二步，回到项目根目录`gin-photo-storage/`，为项目编写一个启动脚本。在项目根目录下新建文件`bootstrap.sh`：

```shell
#!/bin/sh

# 当本脚本被执行时
# 用sed命令将server.conf里的配置项替换成环境变量的值
sed -i "s/<SERVER_PORT>/$SERVER_PORT/g" conf/server.conf
sed -i "s/<DB_HOST>/$DB_HOST/g" conf/server.conf
sed -i "s/<REDIS_HOST>/$REDIS_HOST/g" conf/server.conf
sed -i "s/<ES_HOST>/$ES_HOST/g" conf/server.conf

# 把传进来的参数当作命令执行，并用它替换掉当前进程
exec "$@"
```
第三步，为项目生成依赖管理的文件。由于我的Golang版本是1.11，所以可以使用go module来管理第三方package。在项目根目录`gin-photo-storage/`下，生成一个`go.mod`文件：

```
go mod init gin-photo-storage
```
目录下会自动生成一个`go.mod`文件。**（建议Golang版本较低的读者也升级到 >= 1.11的版本，使用go module管理第三方package）**

第四步，开始编写项目的Dockerfile，用来制作项目的image。在根目录`gin-photo-storage/`下新建`Dockerfile`：

```
FROM golang:latest

WORKDIR /web/gin-photo-storage  # 设定container内的工作目录

COPY . .    # 把当前目录下所有文件copy到container内的工作目录

RUN go build    # 在container内编译，生成go-photo-storage可执行程序

CMD ["./gin-photo-storage"]     # 把./gin-photo-storage当作参数传入bootstrap.sh

ENTRYPOINT ["./bootstrap.sh"]   # 执行bootstrap.sh脚本
```
第五步，有了`Dockerfile`，可以制作自定义的image了，在项目根目录`gin-photo-storage/`下执行：

```
docker build -t my-web:v1.0 .
```
**Note：命令最后有个`.`，表示基于当前目录下的所有文件（`Dockerfile`、`db.sql`）。**

第六步，基于刚刚生成的image，启动container：

```
docker run 
    --name gin \ 
    -d \
    --link mysql \
    --link es \
    --link redis \
    -e DB_HOST="mysql" \
    -e REDIS_HOST="redis" \
    -e ES_HOST="es" \
    -e SERVER_PORT="9088" \
    my-web:v1.0
```
**Note：SERVER_PORT也可以不用`9088`，可使用任意空闲端口。**

> 参数说明：
> 
> --name xxx：给container起名叫xxx，可以是任意自定义的名字
> 
> -e xxx=yyy：给container设定环境变量xxx，值为yyy
> 
> -d：后台运行，不要卡死在前台显示
> 
> --link xxx：把当前container链接到xxx container的网络环境中，也就是说对于当前container而言，xxx container的hostname就是xxx，直接ping xxx可测试连通性。

## 部署nginx
当我们成功跑起`MySQL`、`Redis`、`Elasticsearch`、`gin项目`之后，其实必要的服务已经全部启动了。但是由于上一篇教程提到了`nginx`，而且`nginx`在实际项目中还是非常常用的，所以我们最后再部署一个`nginx`，为项目提供反向代理。

第一步，在本机上任意目录下，新建目录`image-nginx`，并把上一篇教程 [《[Gin教程-9]https与nginx部署》](https://marcoma.xyz/2019/03/21/gin-tutorial-9/) 中我们生成的秘钥和证书：`server.key`和`server.crt`复制到`image-nginx`目录下。

第二步，在`image-nginx`目录下新建启动脚本`bootstrap.sh`：

```shell
#!/bin/sh

cat > /etc/nginx/conf.d/my.conf << EOF
events {
     worker_connections  1024;
}

http {
    server {
        listen       443 ssl;
        server_name  my-web;
    
        ssl_certificate      /server.crt;
        ssl_certificate_key  /server.key;
    
        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;
    
    	ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
    
        location / {
    		proxy_pass https://${APP_HOST}:${APP_PORT};
    	}
    }
}
EOF

exec "$@"
```
上述脚本就是把一段`nginx`配置写到container里的`/etc/nginx/conf.d/my.conf`，然后执行传入的参数（一条启动`nginx`的命令）。

第三步，在`image-nginx`目录下新建`Dockerfile`：

```
FROM nginx:1.14-alpine  # 使用alpine镜像，节省空间

COPY server.crt server.key bootstrap.sh /   # 将文件copy到container内的根目录

CMD ["nginx", "-c", "/etc/nginx/conf.d/my.conf", "-g", "daemon off;"]

ENTRYPOINT ["./bootstrap.sh"]   # 启动时执行bootstrap脚本
```
第四步，基于上述`Dockerfile`，在`image-nginx`目录下执行命令构建自定义image：

```
docker build -t my-nginx:v1.0
```
第五步，基于刚刚构建的image，启动为container：

```
docker run \
    --name nginx \
    -d \
    --link gin \
    -p 9999:443 \
    -e APP_HOST="gin" \
    -e APP_PORT="9088" \
    my-nginx:v1.0
```
**Note：APP\_PORT要和上面启动gin container时的SERVER_PORT一样，才能成功把请求转发到gin服务器上。至于-p 9999:443中的9999，则可以随便选择任意可用端口代替。**

> 参数说明：
> 
> -p xxx:yyy：把container内的yyy端口映射到宿主机的xxx端口，意味着访问宿主机的xxx端口即可访问到container内的yyy端口。

## 整合测试
当我们把上述所有container成功启动后，用`docker ps`查看正在运行的container：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-docker-ps.png)

然后我们用`Postman`工具测试一下接口。因为上面我们已经把nginx container的443端口映射到宿主机的9999端口，所以访问`https://localhost:9999/api/v1/......`应该就可以成功调用我们的gin API。

直接试试用户注册接口`AddAuth()`：

![](https://blog-image-1253224514.cos.ap-guangzhou.myqcloud.com/gin-tutorial-docker-test.png)

其他API就不在文中一一测试了。

## 总结

+ Docker container之间看起来非常独立，一个container似乎是一个完全独立的操作系统，不受别人干预。但是多个container确实是运行在同一台主机上的，那这种独立性是怎么实现的呢？它和另外一种方式——虚拟机相比，有什么异同呢？

    + 关键词：LXC、Linux namespace、Linux cgroups

+ 启动container后，我们可以在container里随意新建、删除、修改文件。当把该container关闭之后，用image再重新启动一个container，为什么所有文件和数据都还能保持没被修改过的状态？

    + 关键词：分层构建、联合挂载

+ 本教程中Elasticsearch使用了默认的9200端口，假如我们想改用自定义端口，但是该镜像没有提供启动参数，我们可以怎么改呢？

    + 关键词：自定义Dockerfile、Entrypoint、entrypoint.sh脚本

+ 对于`MySQL`、`gin项目`、`nginx`的镜像，我们都是继承某个base image，然后自定义的。所以制作出来的镜像都存在了本地，别人不能直接pull，有没有办法发布我们自定义的镜像让别人也能使用呢？

    + 关键词：registry、私有registry、CNCF harbor

+ 本教程里启动了5个container：`MySQL`、`Redis`、`Elasticsearch`、`gin项目`、`nginx`。对于每一个container，我们都是手动启停的。当container数量剧增，这样手动操作还是很麻烦。有没有办法写好一个启停的脚本或者流程，让系统自动启停？

    + 关键词：容器编排、docker compose、kubernetes

> 项目地址：[gin-photo-storage-example](https://github.com/AcepcsMa/gin-photo-gallery-example)
