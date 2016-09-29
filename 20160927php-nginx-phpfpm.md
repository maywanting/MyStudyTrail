title: "用 nginx 服务器配置多版本的 php"
date: 2016-09-27 22:28:09
tags:
- PHP
- nginx
- php-fpm
categories:
- experience
---

本文配置的是 centos 系统上，用 nginx 服务器同时支持 php7 与 php5。

> ## 缘起

8 月份的时候我写了一篇吐槽的博客 [记一次项目的开发到部署](http://maywanting.wang/2016/08/20/20160820project-deploy/)， 这里面我说过自己配置过 nginx 对于双版本 php 的支持，所以说这篇文章我本来早就想写了，一直没时间……现在补上。

> ## 必要的安装

自己配的时候，原本系统已经装好了 php5.6 和 nginx，只不过服务都没跑起来，所以我只要装 php7 就行了。那么 php7 我是怎么装的呢，恩……采用最原始的下载 C 代码编译安装…………具体的安装过程下一篇博客记录吧。

这里吐槽一下 ubuntu16.04，我用 apt 包管理器装 php 的时候没注意，等我装完一看装的 php7……瞬间惊呆了。

> ## nginx 与 php 之间的通信

在写怎么配置之前，我觉得有必要说清楚 ngnix 与 php 之间是怎么通信的。让我们一步一步来哈～

首先一个请求过来，例如 `http://php7.host/index.php`，然后 nginx 就根据域名找对应的 server 块，这里域名是 `php7.host`，所以 nginx 会找 server 中设置 `server_name php7.host`，然后解析里面 location 规则。

但凡是支持运行 php 的，必定会有对于请求结尾是 `.php` 的处理，这种处理的 location，基本都是以下的配置规则，或者是它的精细化。

``` bash
server {
    server_name php7.host;

    …………

    location ~ \.php {
        //对于 php 请求的处理
    }
}
```

以上匹配规则就说明最后带 `.php` 的执行以下操作。

实际上，nginx 本身是不支持调用解析 php，所以必须通过一个通用的接口来调起守护进程进行 php 解析，这个就是 FastCGI。所谓 FastCGI，就是在 Http server（例如 nginx）和动态脚本语言（例如 php）中间通信的的接口。它负责启用一个或多个叫做 FastCGI 进程管理器的守护进程来解析 php，而 php-fpm 就是 FastCGI 进程管理器中处理 php 的一种，也是处理 php 最常用的一种，至少目前的 php 都默认把 php-fpm 都一起编译进了内核。

所以 nginx 在处理动态脚本的时候其实就是一个反向代理服务器，自身处理一些静态请求（例如请求文件），然后将所有需要脚本语言解析的动态请求（例如 php，python）全部交给 FastCGI 接口。

FastCGI 进程管理器与 nginx 通信则是通过 socket，文件 socket 和 ip socket 都可以，所以一个 FastCGI 进程管理器就会监听一个 socket 文件或者一个端口，且不能重复占用。

根据以上的知识，其实做到支持两个版本的 php 就是将两个请求的 url 单独分开处理，交给两个不同的 php-fpm 进程进行解析，而且分别使用不同的 socket 通信。

> ## 配置 php5

这里的配置和普通意义上配置 php 环境类似。

### 1、nginx 配置

首先说一下，为了以示区别，php5 不用默认的 `localhost` 访问，采用域名 `php5.host`，然后本地设置 host 指向 `127.0.0.1`，然后单独拉个 server 块，FastCGI 进程管理器指定监听 socket 文件。

以下为具体的 server 块配置，具体在 nginx.conf 配置

``` bash
upstream phpfile{
    server unix:/run/php/phpfpm.socket; //一般安装 php 的时候就会有默认的 phpfpm.socket 文件，指向这个文件就行了。
}

server {
    listen 80;
    server_name php5.host;

    root /data/www/html/php5/;  //解析地址的根目录

    access_log /var/log/nginx/php5_access.log; //访问日志，这个很重要，往往查日志需要这个。
    error_log /var/log/nginx/php5_error.log; //错误日志，这个也很重要，排查 bug 往往需要查看这个

    location / {
        index index.html index.php; //当只有一个 url 过来，指明默认访问的文件。
    }

    location ~ \.php { //所有后缀名为 .php 的都执行以下操作
        fastcgi_pass phpfile; //全部交给 /run/php/phpfpm.socket。
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

然后重启 nginx

``` bash
/usr/local/nginx/sbin/nginx -t;
/usr/local/nginx/sbin/nginx -s reload;
```

### 2、php-fpm 配置

nginx 配置之后，然后配置 php-fpm.conf

主要就是让 php-fpm 从 `/run/php/phpfpm.socket` 中获取数据。

``` bash
[global]
pid = /run/php/php-fpm.pid
error_log = /var/log/php/php5-fpm.log
log_level = notice

[www]
listen = /run/php/phpfpm.socket //通信数据获取来源
listen.backlog = -1
listen.allowed_clients = 127.0.0.1
listen.owner = www-data
listen.group = www-data
listen.mode = 0666
user = www-data
group = www-data
pm = dynamic
pm.max_children = 1024
pm.start_servers = 50
pm.min_spare_servers = 50
pm.max_spare_servers = 1024
request_terminate_timeout = 100
request_slowlog_timeout = 0
slowlog = /var/log/php/php5-slow.log
```

然后启动 php-fpm 就可以了

``` bash
/usr/local/php/sbin/php-fpm --fpm-config /usr/local/php/etc/php-fpm.conf
```

贴个图展示下成果

![php5 配置成功](../../../../img/20160927_2.jpg)

> ## 配置 php7

由于 php7 是后来我编译安装的，原本 php5.6 装在 `/usr/local/php/` 下，所以 php7 为了防止名字冲突（其实已经很多地方已经冲突了，造成我在部署 php7 的时候踩了一个坑），php7 就装在 `/usr/local/php7/` 下

### 1、nginx 配置

和 php5 一样，为了以示区别，php7 用域名 `php7.host` 访问，本地设置 host 指向 `127.0.0.1`，然后也是单独拉出来一个 server 块。这里指定 FastCHI 进程管理器监听一个端口，就 8022 吧。

以下为具体的 server 块配置，具体在 nginx.conf 配置

``` bash
upstream phpport{
    server 127.0.0.1:8002;
}

server {
    listen 80;
    server_name php7.host;

    root /data/www/html/php7/;  //解析地址的根目录

    access_log /var/log/nginx/php7_access.log; //访问日志，这个很重要，往往查日志需要这个。
    error_log /var/log/nginx/php7_error.log; //错误日志，这个也很重要，排查 bug 往往需要查看这个

    location / {
        index index.html index.php; //当只有一个 url 过来，指明默认访问的文件。
    }

    location ~ \.php { //所有后缀名为 .php 的都执行以下操作
        fastcgi_pass phpport; //全部交给 127.0.0.1:8022 这个端口来。
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

然后重启 nginx

``` bash
/usr/local/nginx/sbin/nginx -t;
/usr/local/nginx/sbin/nginx -s reload;
```

### 2、php-fpm 配置

nginx 配置之后，然后配置 php-fpm.conf

主要就是让 php-fpm 从 8022 端口中获取数据。

``` bash
[global]
pid = /run/php7/php-fpm.pid
error_log = /var/log/php7/php7-fpm.log
log_level = notice

[www]
listen = 127.0.0.1:8022 //通信数据获取来源
listen.backlog = -1
listen.allowed_clients = 127.0.0.1
listen.owner = www-data
listen.group = www-data
listen.mode = 0666
user = www-data
group = www-data
pm = dynamic
pm.max_children = 1024
pm.start_servers = 50
pm.min_spare_servers = 50
pm.max_spare_servers = 1024
request_terminate_timeout = 100
request_slowlog_timeout = 0
slowlog = /var/log/php7/php7-slow.log
```

然后启动 php-fpm 就可以了

``` bash
/usr/local/php7/sbin/php-fpm --fpm-config /usr/local/php7/etc/php-fpm.conf
```

贴个图展示下成果

![php7 配置成功](../../../../img/20160927_1.jpg)

> ## 一些吐槽

之前一切都很顺利，在配置 php7 的时候就开始踩坑了。

在装完 php 的时候，php 会默认装一个全局的命令 `php-fpm`，然后这个 `php-fpm` 命令不用想也知道是事先安装的 php5.6 的，php7 的 `php-fpm` 命令则需要通过最原始的绝对路径来调用。不指明路径的话，那么还是 php5.6 的 php-fpm 换了个 php7 配置文件重新跑，而 php7 的服务压根没跑起来。

所以我在配置的时候就发生了诡异的现象：域名 `php5.host` 访问的好好的，启动 php7 的 php-fpm 之后，用 `php7.host` 访问显示的 php 版本还是 php5.6，而 `php5.host` 访问则 404 了。这报错曾让我一度怀疑起了人生……

其实用查看现在跑了哪些进程也可以看出来是否配置正确，因为如果两版本 php 服务都跑起来的话，是会看到两个一毛一样的 php-fpm，当然再加一个参数就可以看出来这两个 php-fpm 分别属于哪种 php 了

``` bash
ps anx | grep php
```

所以说啊，没事别装两个版本的 php……一来一些全局命令如果不注意就会用错版本，二来一些扩展的维护也比较麻烦。

The End~
