---
title: 在Windows下配置多个Git提交账户
date: 2019-07-24 15:02:06
tags: [Git]
categories: #分类(分层)
	- Git
reward: false #是否开启打赏功能
comment: false #是否开启评论功能
top: 1 #置顶层级(数字越大，优先级越高)
repo: FishHunterFree | my-blog #用户名 | 仓库名
src:  #主页摘要缩略图(外链以及相对资源均可)
---
本文记录在Windows下配置两个github账号的过程.
<!--more-->
---

1. 生成并部署SSH Key

安装好Git客户端后，打开git bash，输入以下命令生成user1的SSH Key

```
ssh-keygen -t rsa -C "user1@email.com"
```
在当前用户的.ssh目录下(C:\\Users\\lenovo\\.ssh)会生成id_rsa私钥文件和id_rsa.pub公钥文件，将id_rsa.pub中的内容添加至user1的github中。然后在git bash中输入以下命令测试该用户的SSH密钥是否生效：

```
ssh -T git@github.com 
```
==注：若提示‘ssh: connect to host github.com port 22: Connection timed out’，证明远端仓库拒绝匿名认证。若连接成功则提示‘Hi user1! You've successfully authenticated, but GitHub does not provide shell access.’==


接着生成user2的密钥，注意不能再使用默认的文件名id_rsa，否则会覆盖之前密钥文件

```
ssh-keygen -t rsa -f ~/.ssh/id_rsa2 -C "user2@email.com"
```
再将该用户的公钥文件添加至github中。测试user2的ssh连接时需要指定密钥文件：

```
ssh -T git@github.com -i ~/.ssh/id_rsa2
```

---

2. 配置config文件

在.ssh目录下创建一个config文本文件，每个账号配置一个Host节点。主要配置项说明

```
Host            别名
HostName        主机名
Port            端口
User            用户名
IdentityFile    密钥文件的路径
IdentitiesOnly  只接受SSH Key 登录
PreferredAuthentications publickey  强制使用Public Key验证

# 配置user1 
Host u1.github.com
HostName github.com
IdentityFile ~/.ssh/id_rsa
PreferredAuthentications publickey
User user1

# 配置user2
Host u2.github.com
HostName github.com
IdentityFile C:\\Users\\lenovo\\.ssh\\id_rsa2
PreferredAuthentications publickey
User user2
```
再通过终端测试SSH Key是否生效

```
ssh -T git@u1.github.com
ssh -T git@u2.github.com
```

---

3. 配置用户名及邮箱

如果之前配置过全局的用户名和邮箱，可以按需选择是否取消全局配置。若取消，需要在各仓库下单独配置相应的用户名和邮箱；反之仅需在需要单独用户的仓库下进行配置，未配置的仓库按照全局配置获取。

```
# 全局配置（任意位置执行）
$ git config --global user.name "github's Name"

$ git config --global user.email "github@xx.com"

$ git config --list

# 取消全局配置
git config --global --unset user.name

git config --global --unset user.email

# 局部配置（在下载的项目根目录执行）
$ git config user.name "gitlab's Name"

$ git config user.email "gitlab@xx.com"
```
==注：配置加载的优先策略为先执行局部配置读取，再进行全局配置读取==

---

4. 配置SSH认证私钥
通过puttygen工具（TortoiseGit）生成*.ppk文件,以支持pageant工具(TortoiseGit)认证私钥

---

5. 注意事项
由于本人安装了TortoiseGit,其默认SSH工具为TortoiseGitPlink.exe，且配置了系统环境变量GIT_SSH指向TortoiseGitPlink.exe。在执行hexo d 指令时，默认该SSH工具进行链接，导致失败。后删除系统环境变量，且配置TortoiseGit的SSH工具为Git下的\usr\bin\ssh.exe，解决提交错误的问题