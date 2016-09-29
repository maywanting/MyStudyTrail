title: "git与本地的项目部署"
date: 2015-06-03 22:36:08
tags:
- git
- linux
categories:
- experience
---

本文只是对于新项目的部署给一些参考，并不会给出git的一些详细命令，git的一些详细命令解释man一下全部都有，我这就不废话了。

> ## 初始化git的一些配置

安装了git之后，第一步就是配置用户名和邮箱，可以采用两种方式

### 1、git命令

可以用以下命令直接配置

``` bash
$ git config --global user.name "may"
$ git config --global user.email "maywanting@gmail.com"
```

### 2、修改文件

还有一种则是直接修改配置文件，git的全局配置则是在文件 `~\.gitignore`中,文件的内容是

``` bash
[user]
        name = may
        email = maywanting@gmail.com

```

注意，以上配置只是对于全局，单个项目的配置可以是默认的全局，也可以另外配置

> ## 公钥上传

如果没有公钥的验证，那么每次和github远程连接都需要用户名和密码输入，非常不方便，所以这里就需要公钥。
用命令 `ssh-keygen`生成公钥，然后 ` cat ~/.ssh/id_rsa.pub`,出来的就是公钥，全部复制下来粘贴到github上ssh公钥处就行了。


> ## github上的项目布置到本地

这个非常简单,一个命令就能搞定

```bash
$ git clone URL
```

其中的URL是github上项目的url，github一般会给你三种url，以我的博客url为例：

###https

```bash
https://github.com/maywanting/maywanting.github.io.git
```

这种形式的无论你有没有上传公钥都得输入用户名和密码，相当于网页登录的形式

###ssh

```bash
git@github.com:maywanting/maywanting.github.io.git
```

熟悉linux的应该对ssh都不陌生，采用这种url的话，这时候上传的公钥就有效果了，就不需要输入用户名和密码了。

###subversion

这种的我至今都没有用过，是用在svn上的，既然我们选择了git，那么这里就不多介绍了。

> ## 本地新建项目上传至github

以创建我的博客为例，首先在本地创建一个文件夹`maywanting.github.io/`,然后git初始化。

```bash
$ git init
```

用命令ll一下就会发现目录下面多了一个'.git/'文件。
首先创建一个新的文件，然后将新创建的文件添加到被跟踪的状态，然后提交到本地仓库。

```bash
$ touch README.md
$ git add README.md
$ git commit -m "init commit"
```

然后在github上新建一个空的仓库，将空仓库的ssh复制一下，将本地origin版本代码添加到远程github空仓库上

```bash
$ git remote add origin git@github.com:maywanting/maywanting.github.io.git
```

然后将master分支代码推送到远程仓库

```bash
$ git push origin master
```

一般性的项目会有一个master分支和开发用的dev分支，所以再创建一个dev分支,然后切换到dev分支

```bash
$ git branch dev
$ git checkout dev
```

然后将dev分支推送到远程仓库

```bash
$ git push origin dev
```
这样一套基本的配置就完成了

The End~
