title: "采用PDO对象和mysql进行交互"
date: 2015-05-18 22:28:09
tags: 
- PHP
- sql
- PDO
categories:
- study
---

最近开始自己着手写一个小的项目,采用PHP+Mysql那一套进行编写。所以说，项目一开始便是连接数据库。之前我只了解过通过mysql_connect函数进行连接，这绝对是最低档次的连接数据库，之后知道了一个PDO对象这么个专门处理和数据库之间操作的对象，简直就和发现了新大陆一样。

> ##创建PDO对象

创建PDO对象一般需要三个参数，第一个参数则是要连接数据库的基本信息，包括数据库的IP地址和数据库的名称，第二个参数则是进入数据库的用户名，第三个就是进入密码。

``` php
$dbn = "mysql:host=" . DB_HOST . ";charset=utf8;dbname=" . DB_NAME;
try{

	$db = new PDO($dbn, DB_USER, DB_PASS);
}catch(Exception $e){

	die($e->getMessage());
}
```

- DB_HOST: 数据库的IP地址，本地数据库的话则是`localhost`
- DB_NAME: 数据库的名字
- DB_USER: 登录的用户名
- DB_PASS: 登录密码

> ##SQL语句准备

这里以event表为例，以下为表的结构

``` sql
+-------------+-------------+------+-----+---------------------+----------------+
| Field       | Type        | Null | Key | Default             | Extra          |
+-------------+-------------+------+-----+---------------------+----------------+
| event_id    | int(11)     | NO   | PRI | NULL                | auto_increment |
| event_title | varchar(80) | YES  |     | NULL                |                |
| event_desc  | text        | YES  |     | NULL                |                |
| event_start | timestamp   | NO   | MUL | 0000-00-00 00:00:00 |                |
| event_end   | timestamp   | NO   |     | 0000-00-00 00:00:00 |                |
+-------------+-------------+------+-----+---------------------+----------------+

```

若SQL中带有php中的参数，一种可以直接在以`{$param}`的形式将值传进去，但是这样类型不好控制，还有一种弊病就是容易被SQL注入。所以还有一种则是采用PDO对象附带的参数绑定函数，这样那么对应的SQL语句就需要占位符，好处就是可以在一定程度上避免被sql注入，还可以使用预处理语句，这样运行的效率更高。占位符大致可分为以下两类。

###1、命名参数

需自定义一个字符串作为“命名参数”，每个参数前需要冒号开始

``` php
$sql = "select * from event where `event_id`=:id and `event_title`=:title";

```

###2、问号参数

问号参数一定要和字段的位置顺序对应。

``` php
$sql = "select * from event where `event_id`=? and `event_title`=?";

```

> ##交互操作

前期的准备都已经OK了，那么接下来就开始和数据库进行交互了

###1、准备语句

使用`prepare()`函数准备预处理语句，该函数返回PDO对象。当重复执行一个sql查询，只是参数不一样的时候，这时候使用预处理效力最高，上面写的两种sql语句准备都是在做预处理，占位符的作用就是用来空出参数的位置。

``` php
$stmt = $db->prepare($sql);

```

参数绑定一种可以用到`bindParam()`函数，具体函数的句法如下

`boolean PDOStatement::bindParam(mixed parameter,mixed &variable[,int datatype[,int length[,mixed driver_options]]])；`

- parameter 预处理语句中指定列值占位符的名字
- variable 赋予占位符的引用值
- datatype 显示设置参数的SQL数据类型 
> PDO::PARAM_BOOL：SQL BOOLEAN类型。
> PDO::PARAM_INPUT_OUTPUT：参数传递给存储过程时使用此类型，因此，可以在过程执行后修改。
> PDO::PARAM_INT：SQL INTEGER数据类型。
> PDO::PARAM_NULL：SQL NULL数据类型。
> PDO::PARAM_LOB：SQL大对象数据类型。
> PDO::PARAM_STMT：PDOStatement对象类型，当前不可操作。
> PDO::PARAM_STR：SQL CHAR、VARCHAR和其它字符串数据类型。
- length 指定数据类型长度，只有当赋值为PDO_PARAM_INPUT_OUTPUT才需要
- driver_option 传递任何数据库驱动产徐特定选项

上代码<br/>

- 占位符为命名参数

``` php
$stmt->bindParam(":id", $id, PDO::PARAM_INT);
$stmt->bindParam(":title", $title, PDO::PARAM_STR);

```

- 占位符为问号参数形式

``` php
$stmt->bindParam(1, $id, PDO::PARAM_INT);
$stmt->bindParam(2, $title, PDO::PARAM_STR);

```

还有一种参数绑定的形式则是直接在execute函数里面添加参数列表，详细见下

###3、执行

`execute([array])`方法负责执行准备好的查询，需要替换占位符。一种则是用bindParam函数替换，还有一种则是直接在execute函数中输入array表示。

###4、获取数据

如果执行的是查询语句的话，就得获取从数据库中查询的数据。下面两个函数是我比较常用的。

- fetch() 将结果集中当前记录返回，并将结果指针移至下一条记录，也就是一条一条返回。
函数原型为
`mixed PDOStatement::fetch ([ int $fetch_style [, int $cursor_orientation = PDO::FETCH_ORI_NEXT [, int $cursor_offset = 0 ]]] )`

- fetchAll()  返回所有搜索结果。
函数原型为
`array PDOStatement::fetchAll ([ int $fetch_style [, mixed $fetch_argument [, array $ctor_args = array() ]]] )`
函数返回一个包含结果集中的所有行的二维数组

这两个函数的参数基本相同，下面一一介绍。

- PDO::FETCH_ASSOC: 返回一个索引为结果集列名的数组 
以`select * from event`为例，则搜索的结果var_dump之后为

``` javascript
array(3) {
  [0]=>
  array(5) {
    ["event_id"]=>
    string(1) "1"
    ["event_title"]=>
    string(14) "New Year's Day"
    ["event_desc"]=>
    string(14) "Happy New Year"
    ["event_start"]=>
    string(19) "2015-01-01 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-01 23:59:59"
  }
  [1]=>
  array(5) {
    ["event_id"]=>
    string(1) "3"
    ["event_title"]=>
    string(19) "Middle of the Mouth"
    ["event_desc"]=>
    string(25) "The Middle of the January"
    ["event_start"]=>
    string(19) "2015-01-16 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-16 23:59:59"
  }
}

```

- PDO::FETCH_BOTH(默认): 返回一个索引为结果集列名和以0开始的列号的数组
以`select * from event`为例，则搜索的结果var_dump之后为

``` javascript
array(3) {
  [0]=>
  array(10) {
    ["event_id"]=>
    string(1) "1"
    [0]=>
    string(1) "1"
    ["event_title"]=>
    string(14) "New Year's Day"
    [1]=>
    string(14) "New Year's Day"
    ["event_desc"]=>
    string(14) "Happy New Year"
    [2]=>
    string(14) "Happy New Year"
    ["event_start"]=>
    string(19) "2015-01-01 00:00:00"
    [3]=>
    string(19) "2015-01-01 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-01 23:59:59"
    [4]=>
    string(19) "2015-01-01 23:59:59"
  }
  [1]=>
  array(10) {
    ["event_id"]=>
    string(1) "3"
    [0]=>
    string(1) "3"
    ["event_title"]=>
    string(19) "Middle of the Mouth"
    [1]=>
    string(19) "Middle of the Mouth"
    ["event_desc"]=>
    string(25) "The Middle of the January"
    [2]=>
    string(25) "The Middle of the January"
    ["event_start"]=>
    string(19) "2015-01-16 00:00:00"
    [3]=>
    string(19) "2015-01-16 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-16 23:59:59"
    [4]=>
    string(19) "2015-01-16 23:59:59"
  }
}

```

- PDO::FETCH_BOUND: 返回TRUE，并分配结果集中的列值给mentPDOStatement::bindColumn()方法绑定PHP变量。
以`select `event_id`, `event_title` from event`为例，只不过代码与之前有些不同。

``` php
$stmt->bindColumn('event_id', $id);
$stmt->bindColumn('event_title', $title);

while($stmt->fetch(PDO::FETCH_BOUND))
{
	echo $id . "\t" . $title . "\n";
}

```

以下为输出结果

``` javascript
1	New Year's Day
3	Middle of the Mouth

```

- PDO::FETCH_CLASS: 返回一个请求类的新实例，映射结果集中的列名到类中对应的属性名。如果 fetch_style 包含 PDO::FETCH_CLASSTYPE（例如：PDO::FETCH_CLASS | PDO::FETCH_CLASSTYPE），则类名由第一列的值决定
以`select * from event`为例，则搜索的结果var_dump之后为

``` javascript
array(3) {
  [0]=>
  object(stdClass)#4 (5) {
    ["event_id"]=>
    string(1) "1"
    ["event_title"]=>
    string(14) "New Year's Day"
    ["event_desc"]=>
    string(14) "Happy New Year"
    ["event_start"]=>
    string(19) "2015-01-01 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-01 23:59:59"
  }
  [1]=>
  object(stdClass)#5 (5) {
    ["event_id"]=>
    string(1) "3"
    ["event_title"]=>
    string(19) "Middle of the Mouth"
    ["event_desc"]=>
    string(25) "The Middle of the January"
    ["event_start"]=>
    string(19) "2015-01-16 00:00:00"
    ["event_end"]=>
    string(19) "2015-01-16 23:59:59"
  }
}

```

- PDO::FETCH_NUM: 返回一个索引为以0开始的结果集列号的数组
以`select * from event`为例，则搜索的结果var_dump之后为

``` javascript
array(3) {
  [0]=>
  array(5) {
    [0]=>
    string(1) "1"
    [1]=>
    string(14) "New Year's Day"
    [2]=>
    string(14) "Happy New Year"
    [3]=>
    string(19) "2015-01-01 00:00:00"
    [4]=>
    string(19) "2015-01-01 23:59:59"
  }
  [1]=>
  array(5) {
    [0]=>
    string(1) "3"
    [1]=>
    string(19) "Middle of the Mouth"
    [2]=>
    string(25) "The Middle of the January"
    [3]=>
    string(19) "2015-01-16 00:00:00"
    [4]=>
    string(19) "2015-01-16 23:59:59"
  }
}

```

接下来的我用的不怎么多，简单介绍下
- PDO::FETCH_INFO: 更新一个被请求类已存在的实例，映射结果集中的列到类中命名的属性
- PDO::FETCH_LAZY: 结合使用 PDO::FETCH_BOTH 和 PDO::FETCH_OBJ，创建供用来访问的对象变量名
- PDO::FETCH_OBJ: 返回一个属性名对应结果集列名的匿名对象

###5、释放

`bool PDOStatement::closeCursor ( void )`释放到数据库服务的连接，以便发出其他SQL语句，但使语句处于一个可以被再次执行的状态。
等同于以下代码

``` php
do {
	while ($stmt->fetch());
	if (!$stmt->nextRowset())	break;
} while (true);

```

> ##代码整合

总结上面，写一下完整的过程

###1、带命名参数占位符

``` php
$dbn = "mysql:host=" . DB_HOST . ";charset=utf8;dbname=" . DB_NAME;

try{
	$db = new PDO($dbn, DB_USER, DB_PASS);
}catch(Exception $e){
	die($e->getMessage());
}

$sql = "select * from `event` where `event_id`=:id and `event_title`=:title"; 
$stmt = $db->prepare($sql);

$stmt->bindParam(":id", $id, PDO::PARAM_INT);
$stmt->bindParam(":title", $title, PDO::PARAM_STR);

if ($stmt->execute() == false){	
	//运行报错处理，多半是语法错误
}

$results = $stmt->fetchAll(PDO::FETCH_ASSOC);  //$result存储返回的数据

$stmt->closeCursor();

```

###2、带问号参数占位符

```php
$dbn = "mysql:host=" . DB_HOST . ";charset=utf8;dbname=" . DB_NAME;

try{
	$db = new PDO($dbn, DB_USER, DB_PASS);
}catch(Exception $e){
	die($e->getMessage());
}

$sql = "select * from `event` where `event_id`=? and `event_title`=?"; 
$stmt = $db->prepare($sql);

$stmt->bindParam(1, $id, PDO::PARAM_INT);
$stmt->bindParam(2, $title, PDO::PARAM_STR);

if ($stmt->execute() == false){
	//运行报错处理，多半是语法错误
}

$results = $stmt->fetchAll(PDO::FETCH_ASSOC); //$result存储返回的数据

$stmt->closeCursor();

```

还有一种则是不用bindParam()函数绑定参数，不过这种方法只能使用问号参数占位符，还有啊，个人感觉这种方法会让安全性降低= =

``` php
$dbn = "mysql:host=" . DB_HOST . ";charset=utf8;dbname=" . DB_NAME;

try{
	$db = new PDO($dbn, DB_USER, DB_PASS);
}catch(Exception $e){
	die($e->getMessage());
}

$sql = "select * from `event` where `event_id`=? and `event_title`=?"; 
$stmt = $db->prepare($sql);

if ($stmt->execute(array($id, $title)) == false){
	//运行报错处理，多半是语法错误
}

$results = $stmt->fetchAll(PDO::FETCH_ASSOC); //$result存储返回的数据

$stmt->closeCursor();

```

这就是一套基本的交互过程，不过本文以查询为例，插入和删除也是同样的，只是没有了获取数据这一过程。

The End~
