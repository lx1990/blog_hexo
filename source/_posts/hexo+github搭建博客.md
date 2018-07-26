---
title: hexo+github搭建博客
tags: 
- hexo
categories: 
- 前端
date: 2018-07-26 15:46:00
---
[hexo官方文档](https://hexo.io/zh-cn/docs/)

&emsp;&emsp;前文说到此博客是基于hexo框架+github搭建起来的，下面来进行详细说明搭建过程。
### 在github注册博客地址
>  github上新建一个仓库，命名为XXXX.github.io，xxxx为你的用户名
<!-- more -->
### 搭建hexo
- **安装[Nodejs](https://nodejs.org/en/)**
- **安装[Git](https://git-scm.com/)**
- **安装hexo-cli**
> npm install -g hexo-cli

- **建站**
> hexo init <folder\>
> cd <folder\>
> npm install

### 使用hexo
- **启动hexo**
> hexo server 

&emsp;&emsp;此时，访问localhost:4000，已经可以看到博客了

- **新增一篇文章**
> hexo new <title\>

&emsp;&emsp;此时，在项目内的`source/_posts`目录下会出现一个名为title的md文件，进入编辑即可

- **生成静态文件**
> hexo generate 

&emsp;&emsp;接下来启动服务，可以看到新增的文章已经更新上去了

### 部署(git版本)
&emsp;&emsp;Hexo 提供了快速方便的一键部署功能，让您只需一条命令就能将网站部署到服务器上。
- **安装部署模块**
> npm install hexo-deployer-git --save
- **在`_config.yml`中修改参数**
~~~
deploy:
  type: git
  repository: 你的仓库地址
  branch: master
~~~

- **部署**
> hexo deploy 

&emsp;&emsp;随后，访问你自己的地址xxx.github.io

### 常用命令
- **init**
> hexo init [folder]

&emsp;&emsp;新建一个网站。如果没有设置`folder`，Hexo 默认在目前的文件夹建立网站。

- **new** 
> hexo new [layout] <title\>

&emsp;&emsp;新建一篇文章。如果没有设置 `layout` 的话，默认使用 `_config.yml` 中的 `default_layout` 参数代替。如果标题包含空格的话，请使用引号括起来。

- **generate**
> hexo generate 可简写为：hexo g

- **server**
> hexo server 可简写为：hexo s

- **clean** 
> hexo clean

&emsp;&emsp;清除缓存文件 (`db.json`) 和已生成的静态文件 (`public`)。

- **deploy**
> hexo deploy 可简写为：hexo d

- **文件生成后立即部署网站**
> hexo g -d 或 hexo d -g

----------------**华丽的分割线**----------------

**若未标明转载，本博客内容均为原创。**

**版权归作者所有，转载请注明出处。**

**若有帮助(批评指正)，还请留言，您的建议是我不断前进的动力**
