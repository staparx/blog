+++
author = "Staparx"
title = "基于Hugo搭建个人博客（Stack主题 + GithubPages持续化部署）"
date = "2022-12-09"
description = "第一篇博客，当然是要记录一下这个站点是怎么搭建起来的啦～"
tags = [
    "Blog",
    "Hugo",
    "Github",
    "hugo-theme-stack",
]
categories = [
    "博客搭建",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
image = "pawel-czerwinski-8uZPynIu-rQ-unsplash.jpg"
+++

# 一、基于Hugo搭建个人博客

## 为何选择Hugo

说白了，就是懒。

作为一个后端开发，即想要一个稍微好看的博客网站，又不想在前端上下太大的功夫。

而Hugo静态网站生成，充分满足了我们的需求。  

它不仅解决了Web端环境依赖的问题，并且使用简单、部署方便，支持 **Markdown** 的写作方式，通过 **LiveReload** 实时刷新，简直不要太舒服！


## 安装Hugo

### 有go环境
[Hugo安装官方文档](https://github.com/gohugoio/hugo#install-hugo-as-your-site-generator-binary-install)

根据文档，直接install下来就完事了。
```go
go install github.com/gohugoio/hugo@latest
```
相信你的环境变量应该指向了你的 **$GOPATH/bin**了，直接去检验安装吧。

### 没有go环境
[Hugo - Releases](https://github.com/gohugoio/hugo/releases)

根据自己的操作系统，去下载吧。最后别忘了设置环境变量哦～。

### 检验安装
在命令行输入
```
hugo env -v
```
显示了hugo版本信息就算成功。

## 开始Hugo
1.创建一个新的博客项目

```
hugo new site blog
```

2.添加一个主题

可以自己选择一个喜欢的[Hugo - 主题](https://themes.gohugo.io/)，本站点使用的是[hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack)。
```
//进入项目空间
cd blog 

//初始化git仓库
git init
 
//将主题添加到主模块中
git submodule add git://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack

//修改配置文件的主题设置
echo 'theme = "hugo-theme-stack"' >> config.toml


// 以下针对 **hugo-theme-stack**这个主题进行配置，若使用的其他主题，可忽略后面的命令。

// 把themes/hugo-theme-stack/exampleSite 文件夹中的config.yaml 复制到站点目录下
cp themes/hugo-theme-stack/exampleSite/config.yaml ./config.yaml

//Hugo配置文件会读取config.toml或config.yaml，所以需要把初始化时生成的config.toml删除掉。
rm -rf config.toml

// 可以把博客模版\themes\hugo-theme-stack\archetypes 复制到 \archetypes中
cp themes/hugo-theme-stack/archetypes ./archetypes

```

3.使用主题自带的页面

想要初始化的时候有点页面，可以直接将主题文件内的 **themes/hugo-theme-stack/exampleSite/content** 里面的page和post页面，cp到主站点的content里面。

PS：重启hugo server后，可能会出现超时的问题，是因为部分页面引用了外网链接，将**content/post**里面的这两个文件删除就可以正常跑起来了。
> placeholder-text
> 
> rich-content

4.创建新的博客页面
新博客页面统一放置在**content/post**路径下，使用new命令会根据**archetypes**内预先配置好的模版创建.md文件。
```
hugo new posts/new/index.md
```

5.本地启动服务
```
hugo server -D
```
默认端口为 **:1313**

直接打开http://localhost:1313/，就可以看到通过主题初始化后的博客啦。

6.构建静态页面
```
hugo -D
```
命令执行后，会在站点目录下生成一个 **./public** 文件，所有静态文件都打包在该目录下。

后续部署会根据该文件进行静态文件部署～

##