title: "用nginx服务器配置多版本的php"
date: 2015-05-18 22:28:09
tags:
- PHP
- nginx
- php-fpm
categories:
- experience
---

本文配置的是centos系统上，用nginx服务器同时支持php7与php5

> ## 缘起

8月份的时候我写了一篇吐槽的博客 [记一次项目的开发到部署](http://maywanting.wang/2016/08/20/20160820project-deploy/), 这里面我说过自己配置过nginx对于双版本php的支持，所以说这篇文章我来早就想写了，一直没时间……，现在补上。

> ## 必要的安装

其实自己配的时候，其实已经装好了php5.6和nginx，只不过服务都没跑起来，所以我只要装php7就行了。那么php7是怎么装的呢，恩……采用最原始的下载C代码编译安装…………，具体的安装过程下一篇博客记录吧。

这里吐槽一下ubuntu16.04，我用apt包管理器装php的时候没注意，等我装完一看装的php7……瞬间惊呆了。

> ## nginx与php之间的通信

再写怎么配置之前，我觉得有必要搞清楚ngnix与php之间是怎么通信的。让我们一步一步来哈～

首先一个请求过来，例如`http://php7.host/index.php`, 然后nginx就根据域名找对应的server块，这里域名是`php7.host`，所以nginx会找server中设置 `server_name php7.host`，然后解析里面location规则。

但凡是支持运行php的，必定会有对于.php请求的处理，这种处理的location，会是以下的配置规则，或者是它的精细化。

``` config
server {
    server_name php7.host;

    …………

    location ~ \.php {
        //对于php请求的处理
    }
}
```

以上匹配规则就说明最后带`.php`的执行以下操作。

实际上，nginx本身是不支持调用解析php，所以必须通过一个通用的接口来调起守护进程进行php解析，这个就是FastCGI。所谓FastCGI，就是在Http server（例如nginx）和动态脚本语言（例如php）中间通信的的接口。它负责启用一个或多个叫做FastCGI进程管理器的守护进程来解析php，而php-fpm就是FastCGI进程管理器中处理php的一种，也是处理php最常用的一种，至少目前的php都把php-fpm都一起编译进了内核。

所以nginx在处理动态脚本的时候就是一个反向代理服务器，自身处理一些静态请求（例如请求文件），然后将所有需要脚本语言解析的动态请求（例如php，python）全部交给FastCGI接口。

FastCGI进程管理器与nginx通信则是通过socket，文件socket和ip socket都可以，所以一个FastCGI进程管理器就会监听一个socket文件或者一个端口，且不能重复占用。

根据以上的知识，其实做到支持两个版本的php就是将两个请求的url单独分开处理，交给两个不同的php-fpm进程，而且分别使用不同的socket通信。

> ## 配置php5

这里的配置和普通意义上配置php环境类似。

### 1、nginx配置

首先说一下，为了以示区别，php5不用默认的`localhost`访问，采用域名`php5.host`，然后本地设置host指向`127.0.0.1`，所以单独拉个server块，然后FastCGI进程管理器指定监听socket文件。

以下为具体的server块配置，具体在nginx.conf配置

``` config
upstream phpfile{
    server unix:/run/php/phpfpm.socket; //一般安装php的时候就会有默认的phpfpm.socket文件，指向这个文件就行了。
}

server {
    listen 80;
    server_name php5.host;

    root /data/www/html/php5/;  //解析地址的根目录

    access_log /var/log/nginx/php5_access.log; //访问日志，这个很重要，往往查日志需要这个。
    error_log /var/log/nginx/php5_error.log; //错误日志，这个也很重要，排查bug往往需要查看这个

    location / {
        index index.html index.php; //当只有一个url过来，指明默认访问的文件。
    }

    location ~ \.php { //所有后缀名为.php的都执行以下操作
        fastcgi_pass phpfile; //全部交给 /run/php/phpfpm.socket。
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

然后重启nginx

``` shell
/usr/local/nginx/sbin/nginx -t;
/usr/local/nginx/sbin/nginx -s reload;
```

### 2、php-fpm配置

nginx配置之后，然后配置php-fpm.conf

主要就是让php-fpm从`/run/php/phpfpm.socket`中获取数据。

``` shell
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

然后启动php-fpm就可以了

``` shell
/usr/local/php/sbin/php-fpm --fpm-config /usr/local/php/etc/php-fpm.conf
```

贴个图展示下成果


> ## 配置php7

由于php7是后来我编译安装的，原本php5.6装在`/usr/local/php/`下，所以php7为了防止名字冲突（其实已经很多地方已经冲突了，造成我在部署php7的时候踩了一个坑），php7就装在`/usr/local/php7/`下

### 1、nginx配置

和php5一样，为了以示区别，php7用域名`php7.host`访问，本地设置host指向`127.0.0.1`，所以也是单独拉出来一个server块。这里指定FastCHI进程管理器监听一个端口，就8022吧。

以下为具体的server块配置，具体在nginx.conf配置

``` config
upstream phpport{
    server 127.0.0.1:8002;
}

server {
    listen 80;
    server_name php7.host;

    root /data/www/html/php7/;  //解析地址的根目录

    access_log /var/log/nginx/php7_access.log; //访问日志，这个很重要，往往查日志需要这个。
    error_log /var/log/nginx/php7_error.log; //错误日志，这个也很重要，排查bug往往需要查看这个

    location / {
        index index.html index.php; //当只有一个url过来，指明默认访问的文件。
    }

    location ~ \.php { //所有后缀名为.php的都执行以下操作
        fastcgi_pass phpport; //全部交给 127.0.0.1:8022这个端口来。
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

然后重启nginx

``` shell
/usr/local/nginx/sbin/nginx -t;
/usr/local/nginx/sbin/nginx -s reload;
```

### 2、php-fpm配置

nginx配置之后，然后配置php-fpm.conf

主要就是让php-fpm从8022端口中获取数据。

``` shell
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

然后启动php-fpm就可以了

``` shell
/usr/local/php7/sbin/php-fpm --fpm-config /usr/local/php7/etc/php-fpm.conf
```

贴个图展示下成果

> ## 一些吐槽

之前一切都很顺利，在配置php7的时候就开始踩坑了。

在装完php的时候，php会默认装一个全局的命令`php-fpm`，然后这个php-fpm命令不用想也知道是事先安装的php5.6的，php7的`php-fpm`命令则需要通过最原始的绝对路径来调用，不指明路径的话，那么还是php5.6的进程管理器换了个配置文件重新跑，php7压根没跑起来。

所以我在配置的时候就发生了诡异的现象：域名php5.host访问的好好的，启动php7的php-fpm之后，用php7.host访问显示的php版本还是php5.6，而php5.host访问则404了。这报错让我一度怀疑起了人生……

其实用查看现在跑了哪些进程也可以看出来是否配置正确，因为如果两个php-fpm都跑起来的话，是会看到两个一毛一样的php-fpm，当然再加一个参数就可以看出来这两个php-fpm分别属于哪种php了

``` shell
ps anx | grep php
```

所以说啊，没事别装两个版本的php……一来一些全局命令如果不注意就会用错版本，二来一些扩展的维护也比较麻烦。
