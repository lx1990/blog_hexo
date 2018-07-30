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
&emsp;&emsp;基于我们现在自己服务的体量，发现使用`confd+nginx+etcd+docker`能够满足我们实现无感知升级的需求，且比较轻量。遂，组长让我研究一下`confd`。`nginx`、`ectd`和`docker`这里不做过多的介绍，只关注`confd`。

### 无感知原理
<!-- more -->
&emsp;&emsp;假设我们有一个服务起了两个`docker`镜像，那么我们把这两个镜像的地址都放在`etcd`中，并启动在`confd`进行监测。
&emsp;&emsp;当我们需要对服务进行升级时，我们可以先在`etcd`中删除一个`docker`的地址，这样`confd`会检测到配置有变动，重新生成新的`nginx`配置文件并且重启`nginx`。
&emsp;&emsp;因为`nginx`的重启时间相当短暂，几乎在瞬间完成，所以此时`nginx`指向不到已经删除掉的`docker`地址，我们可以升级完服务，启动新的`docker`，再将新的`docker`服务的地址放回`etcd`中，剩下的老的`docker`服务也以此类推，从而做到无感知升级。

### 运行环境
> centos7

### 安装confd

- 下载+安装
>
    wget https://github.com/kelseyhightower/confd/releases/download/v0.16.0/confd-0.16.0-linux-amd64
    mkdir -p /opt/confd/bin
    mv confd-0.16.0-linux-amd64 /opt/confd/bin/confd
    chmod +x /opt/confd/bin/confd
    export PATH="$PATH:/opt/confd/bin"


&emsp;&emsp;使用前需要创建相应的配置目录

### 配置confd

- 创建配置目录
> mkdir -p /etc/confd/{conf.d,templates}

- 在`/etc/confd/conf.d`下创建配置文件：
> vim /etc/confd/conf.d/cloud.toml

&emsp;&emsp;这个里面是confd生成的nginx配置文件

~~~ 
    # etcd中keys的前缀
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
> vim /etc/confd/templates/cloud.conf.tmpl

&emsp;&emsp;这个里面是nginx的upstream的模板，被`/etc/confd/conf.d/cloud.toml`读取


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

&emsp;&emsp;上述俩http地址为etcd的地址，自行替换即可

### 操作

> etcdctl set /cloud/upstream/192.168.0.98_1337 192.168.0.98:1337

&emsp;&emsp;接下来，当你在对etcd进行操作的时候，如上，confd会进行轮询，发现有所改动，自动生成nginx的配置文件，并重启nginx

## docker化confd
- docker启动命令
>
    docker run --name confd --restart=always -v /etc/confd/conf.d/:/etc/confd/conf.d/ -v /etc/confd/templates/:/etc/confd/templates/ -v /etc/nginx/conf.d/:/etc/nginx/conf.d/ --net=host -d confd  sh -c "/opt/confd/bin/run.sh  '10' 'http://192.168.0.71:2379' "
    10 表示多少秒轮询一次；后面的地址是etcd的地址

- Dockerfile

~~~
FROM nginx:latest
MAINTAINER lx1990
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

----------------**华丽的分割线**----------------

**若未标明转载，本博客内容均为原创。**

**版权归作者所有，转载请注明出处。**

**若有帮助(批评指正)，还请留言，您的建议是我不断前进的动力**