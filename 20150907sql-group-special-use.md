title: "sql语句中group关键字的另类用法"
date: 2015-09-07 16:08:33
tags:
- sql
categories:
- experience
---

> ##背景扯一下

暑假期间去实验室公司实习，然后碰到一个问题，其实也就是查找数据库数据，就是将商品的特定属性找出来，简化一下就是有两张表，一张表记录商品的各种属性，另一张表则是记下商品的图片，表结构简化如下<br/>
注：数据皆为瞎掰

``` sql
#product表
+------+----------+---------+
|   id | name     |   price |
|------+----------+---------|
|    0 | product1 |    3    |
|    1 | product2 |    4    |
|    2 | product3 |    6.7  |
|    3 | product4 |    5.84 |
+------+----------+---------+

#product_image
+------------+--------------+----------------------+
|   image_id |   product_id | image_url            |
|------------+--------------+----------------------|
|          1 |            0 | ./image/food1.jpg    |
|          2 |            0 | ./image/food2.jpg    |
|          3 |            0 | ./image/food3.jpg    |
|          4 |            3 | ./image/food2.jpg    |
|          5 |            2 | ./image/food34.jpg   |
|          6 |            1 | ./image/food3327.jpg |
|          7 |            1 | ./image/food9327.jpg |
|          8 |            3 | ./image/foodff27.jpg |
+------------+--------------+----------------------+
```

> ##问题描述

现在的要求就是查找商品并且附带一张图片出来，这种情况下多半用于所有商品列表的情况，只要一张图片就ok了。<br/>
一开始我写的sql语句是

``` sql
SELECT t1.*, t2.* FROM product AS t1 LEFT JOIN product_image AS t2 ON t1.id=t2.product_id;
```
不难想象以下搜索结果

``` sql
+------+----------+---------+------------+--------------+----------------------+
|   id | name     |   price |   image_id |   product_id | image_url            |
|------+----------+---------+------------+--------------+----------------------|
|    0 | product1 |    3    |          1 |            0 | ./image/food1.jpg    |   # 记录a1
|    0 | product1 |    3    |          2 |            0 | ./image/food2.jpg    |   # 记录a2
|    0 | product1 |    3    |          3 |            0 | ./image/food3.jpg    |   # 记录a3
|    3 | product4 |    5.84 |          4 |            3 | ./image/food2.jpg    |   # 记录d1
|    2 | product3 |    6.7  |          5 |            2 | ./image/food34.jpg   |   # 记录c1
|    1 | product2 |    4    |          6 |            1 | ./image/food3327.jpg |   # 记录b1
|    1 | product2 |    4    |          7 |            1 | ./image/food9327.jpg |   # 记录b2
|    3 | product4 |    5.84 |          8 |            3 | ./image/foodff27.jpg |   # 记录d2
+------+----------+---------+------------+--------------+----------------------+
```
但是明显一些数据重复了，而且我只想要一张图片就够了。

> ##解决方法

一开始我就像试试看加上distinct关键字，也就是如下语句

``` sql
SELECT DISTINCT t1.id, t1.name, t1.price, t2.* FROM product AS t1 LEFT JOIN product_image AS t2 ON t1.id=t2.product_id;
```
 事实就是并没有什么用。。。=。=搜索结果还是上面一样，无法避免重复<br/>
 然而用分组的group by，效果非常的好，语句如下

 ``` sql
select t1.*, t2.* from product as t1 LEFT JOIN product_image as t2 on t1.id=t2.product_id GROUP BY t1.id；
 ```
 结果如下

 ``` sql
+------+----------+---------+------------+--------------+----------------------+
|   id | name     |   price |   image_id |   product_id | image_url            |
|------+----------+---------+------------+--------------+----------------------|
|    0 | product1 |    3    |          1 |            0 | ./image/food1.jpg    |  # 记录a1
|    1 | product2 |    4    |          6 |            1 | ./image/food3327.jpg |  # 记录b1
|    2 | product3 |    6.7  |          5 |            2 | ./image/food34.jpg   |  # 记录c1
|    3 | product4 |    5.84 |          4 |            3 | ./image/food2.jpg    |  # 记录d1
+------+----------+---------+------------+--------------+----------------------+
 ```
和之前只是单纯的left join比起来，就可以看出image_url选取的是相同条件下第一条记录。

The End~
