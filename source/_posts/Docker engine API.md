---
title: Docker engine API
date: 2018-07-26 23:47:30
tags:
- docker
categories:
- 部署
---
[Docker engine API官方文档](https://docs.docker.com/engine/api/v1.35/)
### 背景介绍
&emsp;&emsp;每次版本升级，都需要手动打镜像，推送仓库，拉取镜像，启动docker。。。需要敲很多的命令，比较繁琐。于是，就开始研究docker engine api，以便之后能够使用接口来替代手敲命令，同时也可升级为可视化的界面，方便部署。
### docker版本
<!-- more -->
~~~
Client:
 Version:       17.12.0-ce
 API version:   1.35
 Go version:    go1.9.2
 Git commit:    c97c6d6
 Built: Wed Dec 27 20:10:14 2017
 OS/Arch:       linux/amd64

Server:
 Engine:
  Version:      17.12.0-ce
  API version:  1.35 (minimum version 1.12)
  Go version:   go1.9.2
  Git commit:   c97c6d6
  Built:        Wed Dec 27 20:12:46 2017
  OS/Arch:      linux/amd64
  Experimental: false
~~~

### 相关配置
- 打开docker的配置
> vim /usr/lib/systemd/system/docker.service

- 连接镜像仓库&&配置docker源地址
>
    添加如下参数：
    ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock --insecure-registry 192.168.0.15:80 --insecure-registry xxx.xxx.xxx.xxx --registry-mirror http://hub-mirror.c.163.com --registry-mirror https://registry.docker-cn.com --registry-mirror https://docker.mirrors.ustc.edu.cn

- 重启docker
>
    systemctl daemon-reload
    service docker restart

### 使用api

- 你的访问域名就是添加了参数的主机地址

- 接口使用postman已调通，

- 需要注意一点：认证问题

> /auth接口，认证的时候不会返回"IdentityToken"

> 在create image的时候(从仓库拉取镜像到本地)，在header中加入X-Registry-Auth参数

~~~
X-Registry-Auth的值为：
{
  "username": "admin",
  "password": "Admin123",
  "serveraddress": "192.168.0.15:80"
}
上方json对象的base64转码
~~~

未完待续
----------------**华丽的分割线**----------------

**若未标明转载，本博客内容均为原创。**

**版权归作者所有，转载请注明出处。**

**若有帮助(批评指正)，还请留言，您的建议是我不断前进的动力**