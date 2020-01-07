---
title: Hexo博客框架下针对Mellow主题进行Gitment评论功能配置
tags:
  - [Mellow]
categories:
  - Hexo
reward: false
comment: true
top: 1
repo: FishHunterFree | my-blog
date: 2020-01-07 15:55:28
src:
---
本文记录Hexo博客框架下针对Mellow主题进行Gitment评论功能配置注意事项
<!--more-->

---
### 使用场景

在个人博客上通过Gitment完成评论的搭建
> (Gitment实现评论的记录是通过GitHub仓库的Issue来完成)

### 操作步骤

1.注册 OAuth Application

首先[点击这里](https://github.com/settings/applications/new)注册自己OAuth Application,此处有四个内容：

```
Application name：          权限应用名称(随意填写)
Homepage URL:               授权主页的全路径(随意填写)
Application description:    授权应用描述(随意填写)
Authorization callback URL: 填写你的博客网址(例:https://user.github.io/blog/),如果有域名则填域名.
```
申请成功后可以看到oauth的client_id,client_secret

2.安装Gitment

```
$ npm install --save gitment
```

3.修改主题配置

此处和个人博客使用的主题相关,本人使用Mellow主题,在主题配置中完成配置后,部署依然会错误,点击登录时报Object ProgressEvent,原因是主题服务器过期,且主题作者未更新,需要在主题文件夹下寻找gitment或者comment的js文件,修改服务器配置.具体改动如下:


```
// 主题配置文件:\MyBlog\themes\mellow\_config.yml
# gitment评论系统
## https://github.com/imsun/gitment
gitment:
  enable: true #是否开启评论系统
  lazy: true #是否开启初始化隐藏评论
  owner: FishHunterFree #GitHub账号名称,就是登陆后的账号名称,Sign in as XXX 的 XXX
  repo: gitment #留言仓库名称,就是需要在本人的GitHub下新建一个Repositories用于记录留言信息
  oauth: 
    client_id: XXX #oauth的client_id
    client_secret: XXX #oauth的client_secret
  perPage: 10 #分页
  
// 在Mellow的文件下找了好久,最后从页面报错入手找到了异常代码位置
// MyBlog\themes\mellow\source\js\plugins\gitmint.browser.js
// 检索https://service.lujingtao.com/gitment,并将其替换成了网络上一个有效的地址https://auth.baixiaotu.cc
```

4.初始化评论插件

点击登入后，(未开放评论)的地方会显示一个按钮让你初始化，点击按钮然后你就可以进行评论,在博客中引入评论,需要调整文件头

```
---
title: title #标题
date: 2017-11-07 09:56:27 #创建时间
tags: [tag1, tag2] #标签(同级)
categories: #分类(分层)
    - cat1
    - cat2
    - cat3
reward: false #是否开启打赏功能
comment: false #是否开启评论功能
top: 1 #置顶层级(数字越大，优先级越高)
repo: codefine | hexo-theme-mellow #用户名 | 仓库名
src: //i.loli.net/2017/12/12/5a2fd18a74471.jpg #主页摘要缩略图(外链以及相对资源均可)
---
```
[更多主题参考可见](https://github.com/codefine/hexo-theme-mellow/wiki/6.-%E6%96%87%E4%BB%B6%E5%A4%B4%E5%B8%B8%E7%94%A8%E8%AE%BE%E7%BD%AE)

