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

- 需要注意一点：认证问题

> /auth接口，认证的时候"IdentityToken"返回值为空(我尝试过，但是没有成功返回token)

> 在create image的时候(从仓库拉取镜像到本地)，在header中加入X-Registry-Auth参数,下面会详细说明



### 具体接口说明
#### Authentication
> method：POST
> /auth

请求参数：
~~~
    {
        "username": "hannibal",
        "password": "xxxx",
        "serveraddress": "https://index.docker.io/v1/"
    }
~~~

#### Images
- List images（列出本地所有镜像）
> method：GET
> /images/json?all=true

- Create an image（从镜像仓库拉取镜像）
> method：POST
> /images/create?fromImage=镜像名字

&emsp;&emsp;Header：
~~~
X-Registry-Auth:下方json对象整体的base64转码
    {
        "username": "hannibal",
        "password": "xxxx",
        "serveraddress": "https://index.docker.io/v1/"
    }
~~~

- Remove an image（删除镜像）
> method：DELETE
> /images/{Image name or ID}

- Tag an image（给镜像打tag）
> method：POST
> /images/{Image name or ID}/tag

- Push an image（推送镜像至仓库）
> method：POST
> /images/{Image name or ID}/push

#### Containers
- List containers（列出所有容器）
> method：GET
> /containers/json?all=true

- Create a container（创建容器）
> method：POST
> /containers/create?name=给容器起的名字

请求参数：
~~~
{
    "Env":[
        "CLOUD_PORT=4000",
        "CLOUD_APP_NAME=cloud"
    ],
    # 启动命令必须按照这样的格式写，不然会报错
    "Cmd": ["pm2", "start" ,"app.js" ,"-i", "2"],
    "Image": "镜像名称",
    "Tag": "latest",
    # Volumes+HostConfig中的Binds等同于：-v
    "Volumes": {
        "/usr/src/app/config/local.js": {},
        "/usr/src/logs": {},
        "/etc/localtime": {}
    },
    "HostConfig": {
        "Binds": [
            "/home/cloud_new/local.js:/usr/src/app/config/local.js",
            "/home/cloud_new/logs/:/usr/src/logs",
            "/etc/localtime:/etc/localtime:ro"
        ],
        # 等同于：–restart=unless-stopped
        "RestartPolicy":{
            "Name": "unless-stopped"
        },
        # 等同于：–net=host
        "NetworkMode":"host"
    }
}
~~~

- Remove a container（删除容器）
> method：DELETE
> /containers/{ID or name of the container}

- Start a container（启动容器）
> method：POST
> /containers/{ID or name of the container}/start

- Stop a container（停止容器）
> method：POST
> /containers/{ID or name of the container}/stop

- Restart a container（重启容器）
> method：POST
> /containers/{ID or name of the container}/restart

- Kill a container（停止容器）
> method：POST
> /containers/{ID or name of the container}/kill

----------------**华丽的分割线**----------------

**若未标明转载，本博客内容均为原创。**

**版权归作者所有，转载请注明出处。**

**若有帮助(批评指正)，还请留言，您的建议是我不断前进的动力**