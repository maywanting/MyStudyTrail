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

实际上，nginx本身是不支持调用解析php的进程，所以必须通过一个通用的接口来调起守护进程进行php解析，这个就是FastCGI。所谓FastCGI，就是在Http server（例如nginx）和动态脚本语言（例如php）中间通信的的接口。它负责
> ## nginx配置

这里为了区别我装的两个版本的php，所以给php5解析的用域名`php5.host`，php7解析则用`php7.host`域名。

### 1、nginx与p

> ## php-fpm配置

> ## 写在最后
