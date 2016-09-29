title: "linux 服务器配置"
date: 2016-07-31 19:50:37
tags:
- linux
- config
categories:
- experience
---

前段时间主管给了我台服务器，感觉就是一个新鲜的肉包子等着我打理，嘿嘿嘿～

所以记录下来整个配置的过程，之后可以写个一键脚本安装也是很方便的。

> ##查看基础信息

主管给我我root账号和ip地址，然后让我随便折腾吧，我等进去之后，第一件是就是查看这个服务器的基本信息，不然连啥类型的linux就开始动手，虽然我也不太清楚各种linux之间的细微区别，但是问题查资料的时候也很方便呢。

``` bash
> uname -a
< Linux localhost.localdomain 2.6.32-642.1.1.el6.x86_64 #1 SMP Tue May 31 21:57:07 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
```

当然也可以查看服务器的版本文件来获取具体信息

``` bash
> cat /proc/version
< Linux version 2.6.32-642.1.1.el6.x86_64 (mockbuild@worker1.bsys.centos.org) (gcc version 4.4.7 20120313 (Red Hat 4.4.7-17) (GCC) ) #1 SMP Tue May 31 21:57:07 UTC 2016
```

嗯～看来是red hat，还是64位操作系统，内核还是2.6……，有点老……

> ##新增用户

登上服务器之后，然后创个属于自己的用户，这样别人登录的时候，不至于因为root打扰到别人，虽然我看了一下记录总共有八个其他的用户，但是最近一次其他用户登上来，也是6月二十几号的事，看了一下安装的软件，啥都没有，这服务器看来真的没有人玩啊。

首先创建用户may

``` bash
adduser may
```

其次是密码

``` bash
passwd *
```

嘿嘿～我的密码怎么可能贴出来。不过在设密码的时候，发现它会检查密码的长度以及是否回文。不过后来在我的强制重试三次，设了密码，此密码超级短，还是回文，看来这个提示也只是看看。

> ##将 may 加入 root 组

``` bash
usermod -g root May
```

这样就可以把 may 这个用户加入到 root 组了，不过还可以设置sudo不用输入密码，这在之后的文章再介绍吧。

> ##更改bash

进入服务器的时候，bash则是最常用的 bash bash，不过我用的最舒服的还是 oh-my-zsh，所以肯定要改。

oh-my-zsh 是基于 zsh 的，所以首先装 zsh

``` bash
yum install zsh
```

然后装 oh-my-zsh

``` bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

接下来我不太喜欢拘泥于一种主题，所以设为随机

``` bash
cd ~
vim .zshrc

ZSH_THEME="random"
```

然后就是给 oh-my-zsh 安装插件，我常用的插件就以下三个

> - colored-man-pages man出来的文档会有颜色标注，这样找命令还是很方便的
> - sudo 这个插件是当一个命令需要 sudo 时，按两下 `Esc` 键，会在开头自动补上 sudo，不需要移动光标，还是很便利的
> - zsh-syntax-highlighting 这个插件会用颜色提示你命令是否存在，红色表示不存在，绿色表示存在，下划线则表示这个目录存在，在命令提示上还是很便利的。

由于zsh-syntax-highlighting这个插件不是oh-my-zsh自带插件，得去github上下载。

``` bash
cd ~/.oh-my-zsh/plugins/
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git

vim ~/.zshrc

plugins=(git colored-man-pages sudo zsh-syntax-highlighting)
```

> ##安装k-vim

vim是我最主要的编程工具，所以肯定得装这个

``` bash
git clone https://github.com/MikeCoder/k-vim.git
```

下载完之后进入目录，`source install.sh`，然后可以去泡杯咖啡了。

> ##更改`Esc`键位与`Caps Lock`键位

为啥要改这两个呢，首先`Caps Lock`这个键就是大小写锁定，用的不多，而且一般大写我都习惯用`Shift`加字母来。其次，由于是vim党，所以频繁使用`Esc`键，手够不到，不方便，然后就把这两个键改了。

主要用`xmodmap`这个命令进行改键位。首先创个文件，比如我这里起名`.keymaps`，然后编辑

``` bash
!
!Swap Caps_Lock and Escape
!
remove Lock = Caps_Lock
keysym Escape = Caps_Lock
keysym Caps_Lock = Escape
add Lock = Caps_Lock
```

然后 `xmodmap .keymaps`键位就交换了。由于这个键位再重启了计算机之后就会复原，所以建议加入开机启动计划。

> ##将密码登录改成公钥登录

每次 ssh 到服务器然后输入密码很是麻烦，所以采用公钥认证登录，这样就不需要输入密码了。

首先在本地生成密钥对

``` bash
ssh-keygen -t rsa -P ""
```
然后进入.ssh目录里面，里面应该有三个文件`id_rsa`, `id_rsa.pub`, `known_hosts`。这里面`id_rsa`为私钥，而`id_rsa.pub`为公钥，然后将公钥复制到服务器上，并将公钥的内容添加到`~/.ssh/authorized_keys`文件中，没有这个文件就创个。

如果这个服务器就你一个人用的话，建议弄成不允许密码登录，这样可以防止别人暴力破解你的服务器密码

在服务器的`/etc/ssh/sshd_config`的文件中，改成如下

``` bash
#PasswordAuthentication yes

改为

PasswordAuthentication no
```

> ## 碎碎念

初步的部署就是这样，反正每当给我一个服务器，我基本都要干这些，记录下来免得自己找东找西找命令。哈哈～

The End~
