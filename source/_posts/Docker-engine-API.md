---
title: Docker engine API
date: 2018-07-26 18:51:30
tags:
---
# Docker engine API
[Docker engine API官方文档](https://docs.docker.com/engine/api/v1.35/)
### docker版本

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
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock --insecure-registry 192.168.0.15:80 --insecure-registry 218.94.141.74:5001 --registry-mirror http://hub-
mirror.c.163.com --registry-mirror https://registry.docker-cn.com --registry-mirror https://docker.mirrors.ustc.edu.cn

- 重启docker

>
systemctl daemon-reload 
service docker restart  

### 使用api

- 你的访问域名就是添加了参数的主机地址
  
- 接口使用postman已调通，作为附件上传

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
