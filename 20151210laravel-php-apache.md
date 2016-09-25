title: "关于laravel配置时遇到的一个坑--rewrite重写"
date: 2015-12-10 21:53:39
tags:
- laravel
- PHP
- apache
- rewrite
categories:
- experience
---

近期接手了学院的一个项目，叫毕业设计管理系统。顾名思义就是用来管理我们院的毕业设计的。于是就趁这个项目学习一下lavarel这个框架。

> ##问题描述

其实说到底还是apache 重写服务没开的锅= 。=

问题呢是这样的，我在配route的时候,配了下面两个路由

``` php
Route::get('login', function() {
	return view('login');
})；

Route::get('/', function() {
	return view('welcome');
});
```
然后打开浏览器输入`localhost/test/public/login`,然后浏览器就说找不到了。但是输入`localhost/test/public`，就可以打开正确的页面。

一开始我以为我的哪里语法啊配置啊啥的写错了，后来在刘X大神的提示下，才发现是apache的rewrite重写没开。

> ##解决方法

解决方法有两个，一个是添加`index.php`,这样就可以找到。例如上面的login页面改成如下

``` php
Route::get('index.php/login', function() {
	return view('login');
});
```
然后浏览器输入`localhost/test/public/index.php/login`就可以找到相应的页面。
当然，这种方法治标不治本，所以开启apache的rewrite功能并配置是非常必要的。

> ##apache的rewrite模块开启

在终端输入命令

``` bash
sudo a2enmod rewrite
```
就开启了重写的功能，顺带一提关闭重写是

``` bash
sudo a2dismod rewrite
```
打开`/etc/apache2/mods-enabled/`这个目录就会发现目录下多了一个

``` bash
rewrite.load -> ../mods-available/rewrite.load
```
这就说明rewrite服务已经开启了。

然后打开`/etc/apache2/apache2.conf`文件里，在配置localhost的xml标签下，添上

``` xml
<Directory {localhost的绝对路径}/test>
        Options FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
</Directory>

```
添加完后，敲`service apache2 restart`命令重启apache就ok了。在此感谢唐XX大神的帮助和纠正！

> ##关于rewrite

> rewrite的主要功能就是实线url的跳转，它的正则表达式是基于perl语言。而且实现有两种方式，一种是基于服务器（httpd.conf）和目录级的（.htacess）。

看了很多博客大家都是这么说的= =，其实说白了就是有两种实现方式，一种则在服务器那里写规则，还有一种就是在具体项目文件里面写.htacess文件来写规则，而具体的规则匹配则是用perl语言版的正则表达式。

很明显laravel框架用的是后者，而且public文件夹下就已经配好了.htacess文件，所以只要服务器开启了rewrite功能，还有指明哪个路径允许重写就行。

以下是laravel框架里的.htacess文件的rewrite配置

``` xml
# Redirect Trailing Slashes If Not A Folder...
RewriteCond %{REQUEST_FILENAME} !-d  
RewriteRule ^(.*)/$ /$1 [L,R=301]  

# Handle Front Controller...
RewriteCond %{REQUEST_FILENAME} !-d #请求的路径不存在
RewriteCond %{REQUEST_FILENAME} !-f #请求的文件不存在
RewriteRule ^ index.php [L]  #满足上述条件则跳转到index.php
```
上述的规则表示，请求的文件或路径不存在则跳转到index.php，比如说请求的是`localhost/test/login`,这个在文件系统中是找不到的，于是就跳转到`localhost/test/index.php`这个文件。然而获取`$_SERVER['REQUEST_URI']`里后面的login这个参数，交给route处理。

也就说，主要输入的路径在实际的文件系统中找不到，就一律跳转到index.php,至于后面具体的哪个页面，则交给route来判断。

所以说啊，这个框架让我惊艳的地方之一就在于route这个神奇东西。

The End~
