---
title: gitlab+gitlab-runner实现自动打包docker镜像
date: 2018-07-31 22:12:49
tags:
- gitlab
- gitlab-ci
- gitlab-runner
- docker
categories:
- 自动化部署
---

[gitlab官网](https://docs.gitlab.com/ee/README.html)

### 获取gitlab的docker镜像
&emsp;&emsp;注意安装的是ce版还是ee版，我们这里使用docker(ce汉化版)进行安装。
<!-- more -->

- 官方版
> docker pull gitlab/gitlab-ce

- [汉化版](https://github.com/beginor/docker-gitlab-ce)
> docker pull beginor/gitlab-ce:11.0.4-ce.0

### 运行
&emsp;&emsp;先建三个文件夹，存放gitlab的配置，数据，日志
>
    mkdir -p /home/gitlab/config
    mkdir -p /home/gitlab/logs
    mkdir -p /home/gitlab/data

- docker启动命令
~~~
    docker run --detach \
        --hostname 192.168.0.15 \
        --publish 8443:443 --publish 880:880 --publish 822:22 \
        --name gitlab \
        --restart unless-stopped \
        --volume /home/gitlab/config:/etc/gitlab \
        --volume /home/gitlab/logs:/var/log/gitlab \
        --volume /home/gitlab/data:/var/opt/gitlab \
        beginor/gitlab-ce:11.0.1-ce.0
~~~

&emsp;&emsp;这里修改了gitlab的内部服务端口，否则在创建新账号后发送的邮件中的修改密码的地址端口不正确，无法访问（可能还有其他地方会受影响），如何修改内部的端口请见下方。

### 修改配置
&emsp;&emsp;每次修改完配置后记得docker restart gitlab
> vim /home/gitlab/config/gitlab.rb

- 修改端口
>
    external_url 'http://192.168.0.15:880'
    nginx['listen_port'] = 880

- 设置邮箱
~~~
    ### Email Settings
     gitlab_rails['gitlab_email_enabled'] = true
     gitlab_rails['gitlab_email_from'] = 'lx1990@xxxx.xxx'
     gitlab_rails['gitlab_email_display_name'] = 'lx1990'
     gitlab_rails['gitlab_email_reply_to'] = 'lx1990@xxxx.xxx'
     gitlab_rails['gitlab_email_subject_suffix'] = ''

     gitlab_rails['smtp_enable'] = true
     gitlab_rails['smtp_address'] = "smtp.ym.163.com"
     gitlab_rails['smtp_port'] = 25
     gitlab_rails['smtp_user_name'] = "lx1990@xxxx.xxx"
     gitlab_rails['smtp_password'] = "xxxxxx"
     gitlab_rails['smtp_domain'] = "xxx.xxx"
     gitlab_rails['smtp_authentication'] = "login"
     gitlab_rails['smtp_enable_starttls_auto'] = true
     gitlab_rails['smtp_tls'] = false
~~~

&emsp;&emsp;这里使用的是网易的企业邮箱，其他邮箱具体设置方法请看文档

[其他邮箱设置](https://docs.gitlab.com/omnibus/settings/smtp.html#doc-nav)

- 登录地址
> 192.168.0.15:880

- 登录默认密码
> 5iveL!fe

&emsp;&emsp;第一次登陆会让你直接修改密码，用户名为`root`

### 安装运行gitlab-runner
- 安装
> docker pull gitlab/gitlab-runner

- 启动
~~~
    docker run -d --name gitlab-runner --restart always \
        -v /home/gitlab-runner/config:/etc/gitlab-runner \
        -v /var/run/docker.sock:/var/run/docker.sock \
        gitlab/gitlab-runner:latest
~~~

- 注册
> docker exec -it gitlab-runner gitlab-ci-multi-runner register

1.输入 GitLab 实例 URL:
>
    Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
    http://192.168.0.15:880

2.输入获取到的用于注册 Runner 的 token:
>
    Please enter the gitlab-ci token for this runner
    xxx
    
- Shared Runners(一个runner可以负责多个项目)

![share token](/images/share-runner.png)
- Specific Runners（一个runner只能负责对应的项目）

![spec token](/images/spec-runner.png)

3.输入该 Runner 的描述，稍后也可通过 GitLab's UI 修改:
>
    Please enter the gitlab-ci description for this runner
    [hostame] my-runner

4.给该 Runner 指派 tags, 稍后也可以在 GitLab's UI 修改:
>
    Please enter the gitlab-ci tags for this runner (comma separated):
    my-tag,another-tag

5.Enter the Runner executor:
>
    Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
    docker

6.如果你选择 Docker 作为你的 executor，注册程序会让你设置一个默认的镜像， 作用于 .gitlab-ci.yml 中未指定镜像的项目：
>
    Please enter the Docker image (eg. ruby:2.1):
    docker:stable


7.修改gitlab-runner的注册信息
> vim /home/gitlab-runner/config/config.toml

~~~
concurrent = 1
check_interval = 0

[[runners]]
  name = "cloud"
  url = "http://192.168.0.15:880/"
  token = "83ad8c9b6d00417e05e5715d686dd6"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "docker:stable"
    privileged = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    cache_dir = "cache"
    shm_size = 0
  [runners.cache]

~~~
&emsp;&emsp;下面，我们需要当我们给git打标签的时候（触发条件，可以设成别的），自动打包docker镜像并推送到镜像仓库中。

### gitlab-ci配置
- 在项目中增加一个`.gitlab-ci.yml`文件
~~~
    image: docker:stable

    services:
      - docker:dind

    before_script:
      - docker login -u xxxx -p xxxx 192.168.0.15:80

    build:
      stage: build
      script:
        - pwd
        - mv ./Dockerfile ../
        - cd ../
        - docker build -t 192.168.0.15:80/xxx/xxx:$CI_COMMIT_REF_NAME .
        - docker images
        - docker push 192.168.0.15:80/xxx/xxx:$CI_COMMIT_REF_NAME
      only:
        - tags
      tags:
        - cloud
~~~
&emsp;&emsp;至此，我们就可以实现打了tag自动生成docker镜像并推送到镜像仓库，结合之前的docker engine api的调用来方便地部署镜像

----------------**华丽的分割线**----------------

**若未标明转载，本博客内容均为原创。**

**版权归作者所有，转载请注明出处。**

**若有帮助(批评指正)，还请留言，您的建议是我不断前进的动力**

