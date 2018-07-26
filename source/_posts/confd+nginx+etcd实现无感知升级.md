---
title: confd+nginx+etcd实现无感知升级
date: 2018-07-26 17:48:28
tags:
- nginx
- confd
- etcd
categories: 
- 运维
---
[confd官网](http://www.confd.io/)
### 背景介绍
&emsp;&emsp;基于我们现在自己服务的体量，发现使用`confd+nginx+etcd+docker`能够满足我们实现无感知升级的需求，且比较轻量。遂，组长让我研究一下`confd`。`nginx`、`ectd`和`docker`这里不做过多的介绍，只关注confd。

### 运行环境
> centos7
<!-- more -->
### 安装confd
- 下载
> 
    wget https://github.com/kelseyhightower/confd/releases/download/v0.16.0/confd-0.16.0-linux-amd64
    mkdir -p /opt/confd/bin
    mv confd-0.16.0-linux-amd64 /opt/confd/bin/confd
    chmod +x /opt/confd/bin/confd
    export PATH="$PATH:/opt/confd/bin"
    
- 下载：

~~~
wget https://github.com/kelseyhightower/confd/releases/download/v0.16.0/confd-0.16.0-linux-amd64
~~~

- 安装

>
    mkdir -p /opt/confd/bin
    mv confd-0.16.0-linux-amd64 /opt/confd/bin/confd
    chmod +x /opt/confd/bin/confd
    export PATH="$PATH:/opt/confd/bin"

使用前需要创建相应的配置目录

### 配置confd

- 创建配置目录
> mkdir -p /etc/confd/{conf.d,templates}

- 在`/etc/confd/conf.d`下创建配置文件：
这个里面是用来生成nginx的配置文件，供nginx使用

> vim /etc/confd/conf.d/cloud.toml

~~~ 
    # keys的前缀
    prefix = "/cloud"
    # src指/etc/confd/templates中的模板文件的名字
    src = "cloud.conf.tmpl"
    # dest指nginx配置文件生成目录
    dest = "/etc/nginx/conf.d/cloud.conf"
    # keys指的是etcd中的keys
    keys = [
        "/subdomain",
        "/upstream"
    ]
    reload_cmd = "nginx -s reload"
    #reload_cmd = "service nginx reload"
~~~

- 在`/etc/confd/templates`下创建模板文件：
这个里面是nginx的upstream的模板，被`/etc/confd/conf.d/cloud.toml`读取

> vim /etc/confd/templates/cloud.conf.tmpl

~~~
    # upstream指从etcd某个键中读取出来的数据
    upstream {{getv "/subdomain"}} {
        {{range getvs "/upstream/*"}}
            server {{.}};
        {{end}}
        }
        server {
            listen      443 ;
            server_name  jswym.com;
            location / {
                proxy_pass        http://{{getv "/subdomain"}};
                proxy_redirect    off;
                proxy_set_header  Host             $host;
                proxy_set_header  X-Real-IP        $remote_addr;
                proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
    }
~~~

- 修改nginx中的配置

>  vim /etc/nginx/nginx.conf 

~~~
    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" "$http_host" '
                        '"$http_user_agent" "$http_x_forwarded_for"'
                        '"$upstream_addr" "$upstream_cache_status" "$upstream_status" "$upstream_response_time"'
                        '"$Cookie_CookKey"';

        access_log  /var/log/nginx/nginx_access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        #keepalive_timeout  0;
        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;   
    }

重点是include这一行
~~~
### 执行生成配置文件

>
    confd -onetime -backend etcd -node http://127.0.0.1:2379    只一次
    confd -interval=60 -backend etcd -node http://127.0.0.1:2379 &   按时间轮询

上述俩http地址为etcd的地址，请自行替换

### 操作

> etcdctl set /cloud/upstream/192.168.0.98_1337 192.168.0.98:1337

接下来，当你在对etcd进行操作的时候，如上，confd会进行轮询，发现有所改动，自动生成nginx的配置文件，并重启nginx

## docker化confd

- docker地址

> docker pull 192.168.0.15:80/service/confd:20180726

- docker启动命令

> docker run --name confd --restart=always -v /etc/confd/conf.d/:/etc/confd/conf.d/ -v /etc/confd/templates/:/etc/confd/templates/ -v /etc/nginx/conf.d/:/etc/nginx/conf.d/ --net=host -d confd  sh -c "/opt/confd/bin/run.sh  '10' 'http://192.168.0.71:2379' " 

> 10 表示多少秒轮询一次；后面的地址是etcd的地址

- Dockerfile

~~~
FROM nginx:latest
MAINTAINER WYM-CLOUD
WORKDIR /opt/confd/bin
COPY ./confd /opt/confd/bin/confd
COPY ./run.sh /opt/confd/bin/run.sh
RUN chmod +x /opt/confd/bin/run.sh
RUN mkdir -p /etc/confd/conf.d
RUN mkdir -p /etc/confd/templates
RUN mkdir -p /etc/nginx/conf.d
~~~

- run.sh

~~~
#!/bin/bash
cd /opt/confd/bin
./confd -interval=$1 -backend etcd -node $2 &
nginx -g 'daemon off;'
~~~