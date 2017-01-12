title: "php中关于mktime的一个坑"
date: 2017-01-12 22:28:09
tags:
- PHP
- mktime
categories:
- study
---

> ## 缘起

在写一个关于时间的问题的代码时有这个一个场景，这个脚本只能在9点半到11点半以及下午1点到3点间运行，其他时间不允许运行

好吧……这微妙的时间我确实在写关于股票盘中的业务

``` php
$time = time();
$amStart = mktime(9, 30); //上午开始时间
$amEnd = mktime(11, 30); //上午结束时间
$pmStart = mktime(13); //下午开始时间
$pmEnd = mktime(15); //下午结束时间

if ($time < $amStart) return false; //开盘前返回false
if ($time > $amEnd && $time < $pmStart) return false; //午休时间
if ($time > $pmEnd) return false; //毕盘结束
```

改正代码

``` php
$time = time();
$amStart = mktime(9, 30); //上午开始时间
$amEnd = mktime(11, 30); //上午结束时间
$pmStart = mktime(13, 0); //下午开始时间
$pmEnd = mktime(15, 0); //下午结束时间

if ($time < $amStart) return false; //开盘前返回false
if ($time > $amEnd && $time < $pmStart) return false; //午休时间
if ($time > $pmEnd) return false; //毕盘结束
```
占个坑提醒自己
