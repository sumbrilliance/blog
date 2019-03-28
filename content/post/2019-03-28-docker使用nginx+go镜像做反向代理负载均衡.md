---
title: "docker多容器实践：nginx+go+beego镜像做反向代理"
date: 2019-03-28
description: ""
coverImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImage: "https://blog-1252872972.cos.ap-chengdu.myqcloud.com/home-bg-o.jpg"
thumbnailImagePosition: "left"
categories:
- docker
tags: 
- go
- nginx
- beego
---

本文通过一个简单的案例实践，描述了多容器多服务下如何对服务配置和关联。具体为

1. 基于 beego实现的一个 http服务，使用 go官方镜像，打包成一个容器1
2. 通过 go 容器，将1中的服务，简单修改，克隆出第二个 http，打包成容器2
3. 使用 nginx 镜像，生成一个反向代理容器，代理到对上述两个容器服务

<!--more-->

<!-- toc -->

# 

# 0x1 准备文件

先为这一波操作新建一个文件夹，里面建立三个文件夹，分别存放 

- http服务1源码

- http服务2源码，

- nginx 配置文件

  例如我的：

  ```
  创建 dockerCompose
  建立：
  beego1/
  beego2/
  nginx/
  ```

  

  ![docker-go1](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/03/docker-go1.png)

  结构：
  ![docker-go2](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/03/docker-go2.png)

  * 请忽略我的 github.com目录下面的无关文件。。
  * 以下把 http服务容器1称为 beego1,http服务容器2称为beego2

# 一、配置 http 服务容器

## 1.我们的http服务使用go开发，所以先拉取go的镜像

> docker pull golang

有了执行环境之后，当然是把代码放到环境中执行。

## 2. 将开发好的 程序放到指定目录下，将其映射到 golang 容器

> cd /path/ 

path 即为刚才我们建的文件夹

既然是http服务，必然要暴露端口给外面访问，先定两个宿主机的端口10086、10087分别映射beego1,beego2端口。beego 库默认使用的端口号是8080，我们使用默认即可。于是创建容器的命令为：

> docker run --name beego1 -p 10086:8080 -v $PWD/beego1/src/:/go/src/ golang bash -c "cd src/beegoDemo/ && go run beegoMain.go"

和

> docker run --name beego2 -p 10087:8080 -v $PWD/beego2/src/:/go/src/ golang bash -c "cd src/beegoDemo/ && go run beegoMain.go"

注意：

* 使用了$PWD环境变量，所以，为什么开始要 cd到对应目录下。
* 为什么使用 ```bash bash -c "cd src/beegoDemo/ && go run beegoMain.go" ``` 呢？

因为go镜像工作目录是 "/go"，进入容器后执行执行 ```go run <绝对路径>/beegoMain.go``` 的话，执行路径将是 "/go" 目录，由于beego的特性，beego会默认在入口文件的同层目录下找 /views 目录下的html 文件，如果执行路径在 "/go"下，将会找不到模板文件，无法显示。

我们的代码路径实际已经映射到 "/go/src/"下，所以期待进入容器时，应先进入项目目录，再执行 go run。所以启动容器时，实际将执行两条命令 "cd src/beegoDemo" 然后是 "go run beegoMain.go"

## 3.测试

分别访问 localhost:10086 和 localhost:10087，看服务是否正常

我们在 /src/beegoDemo/views/index.html 分别写入不同的内容，来查看区别

> cat ./beego1/src/beegoDemo/views/index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello beego1</title>
</head>
<body>
    I am beego1
</body>
</html>
```

> cat ./beego2/src/beegoDemo/views/index.html

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>hello beego2</title>
</head>
<body>
    I am beego2
</body>
</html>%
```



![docker-go3](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/03/docker-go3.gif)

# 二、配置nginx 服务
## 1.拉取镜像
没有指定版本，拉取 latest的，都差不多。

> docker pull nginx

## 2. 编辑代理 

对80端口的访问，代理到 192.168.1.100:10086和192.168.1.100:10087

```
upstream beegos {
	server 192.168.1.100:10086; // 我的ip是192.168.1.100，自行替换为自己的ip
	server 192.168.1.100:10087;
}

server {
    listen       80;
    server_name _;#域名
    location / {
        proxy_pass http://beegos; 
        index index.html index.htm index.php;
    }
}
```

保存为 nginx.conf，存放到开始建立的nginx文件夹下。

## 3. 运行nginx容器

> docker run —name nginxR -v $PWD/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro -p 9999:80 nginx

- 同样用到了$PWD，要注意执行时所在路径

- ro，即只读模式

- 挂载为容器的 nginx/conf.d/default.conf，而不是nginx/nginx.conf

- 将本机的9999 端口映射到容器的 80端口，即通过访问本机的 9999，放问到nginx的80端口服务，nginx再把服务代理到本机的 10086和10087:

  ![docker-nginx1](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/03/docker-nginx1.png)

现在访问 localhost:9999，刷新几次看看

![docker-gong](https://blog-1252872972.cos.ap-chengdu.myqcloud.com/03/docker-gong.gif)

可以看到，刷新的时候，随机访问到了两个容器的服务

# 三、结语

至此，完成了一个简单的利用多个容器实现多容器反向代理负载均衡的案例。多容器协调提供服务是很常见的需求，如果在服务很多的情况下，每个容器单独启动，逐个配置将会是大量的重复劳动，所以，多容器场景下，通常使用docker-compose（当然还有k8s）去做这样的事情。下一篇，我们将用docker-compose来实现一次这次需求。