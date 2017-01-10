title: "关于php中的foreach引用"
date: 2016-12-29 22:28:09
tags:
- PHP
- foreach
- quote
categories:
- study
---

> ##先来看段代码

我在查找(from baidu)关于foreach中引用的资料的时候，百分之九十都是讲的以下代码

``` php
$array = ['one', 'two'];
foreach($array => &$value) {}
print_r($value);

foreach($array => $value) {}
print_r($value);
```

这段代码的输出呢是这个样子的。

``` json
Array
(
    [0] => one
    [1] => two
)
Array
(
    [0] => one
    [1] => one
)
```

出现这个现象的原因就是第一个循环之后，`$value` 则是个引用，指向`$array[1]`，然后下一个循环的一开始，`$array[0]`赋值给`$value`，也就等价于`$array[1] = $array[0]`，然后下一个赋值的时候，就是`$array[1] = $array[1]`。

有没有觉得细思恐惧？解决办法呢就是在下一个循环之前把`$value` unset掉

> ##缘起

为啥要讲楼上这个代码呢，因为自己在找问题的时候，百分之九十都在讲这个…………我也是无语了…………

自己遇到的问题呢是这样的，有一个业务需要请求别人的接口六次，每次获取的数据都不一样，而且呢，这个接口返回的数据量还都挺大的。

第一版代码很简单，就是简单的请求六次，然后从线上跑下来的情况看，这个接口失败的概率很高，基本上五次脚本运行就会有一次异常数据，原因就是接口请求失败

所以呢，我就想将所有的请求都搞成一个数组，如果这次请求失败，那么将这次请求塞到数组末尾，然后再请求。

我知道方法并不是只有这一个……比如说如果这一次请求失败，那么再次请求，请求个五次失败直接脚本退出。

但是我自己就是想这么塞在数组末尾干，所以就出现了以下的问题

> ##初始代码

我一开始的代码可以抽象为这样

``` PHP
$array = [
    ['0', 'api0'],
    ['1', 'api1'],
    ['2', 'api2'],
    ['3', 'api3'],
    ['4', 'api4'],
];

$num = 1;
foreach ($array as $value) {
    if  ($num == 1 && $value[0] == 1) {
        array_push($array, $value);
        var_dump($array);
        $num++;
    }

    if ($num == 2 && $value[0] == 3) {
        array_push($array, $value);
        var_dump($array);
        $num++;
    }

    echo $value[0] . "\n";
}

var_dump($array);
exit;
```

这段代码很好理解，我就是想在把`['1', 'api1']`和`['3', 'api3']`塞在数组后面，预期是想把前面的api都执行一边之后，再执行api1和api3，（PS：实际情况是任意api都可能重新执行一遍，我这里只是为了方便测试所以固定了重新执行api1和api3）

然而实际上的执行结果如下

``` PHP
0
array(6){api0, api1, api2, api3, api4, api1}
1
2
array(7){api0, api1, api2, api3, api4, api1, api3}
3
4
```

api1和api3没有再次执行……，然后觉得很不可思议啊

然后自己瞎折腾的时候，发现了解决办法，也就是foreach的时候换成下面的样子

``` PHP
foreach ($array as &$value) {
```

然后，循环体内一模一样的代码，然后api1和api3就再次执行了，然后这是为什么呢！？

> ## 官方文档的foreach

官方文档中关于![foreach](http://php.net/manual/zh/control-structures.foreach.php)的说明大致就是下面的意思

- 如果`$value`没有加`&`那么会将`$value`的值拷贝一份给$value
- 如果`$value`加了`&`那么就可以直接改`$array`的值，具体咋改就是通过值赋给`$value`就可以改了

然后给了个例子说明如何更改数组

``` php
$arr = array(1, 2, 3, 4);
foreach ($arr as &$value) {
    $value = $value * 2;
}
// $arr is now array(2, 4, 6, 8)
unset($value); // 最后取消掉引用
```

我觉得读到这可以完美解决我一开始看到的问题，但同时如果上面修改的方法，还不如下面的修改代码，而且还免去了引用带来的莫名其妙的值改了的情况

``` php
$arr = array(1, 2, 3, 4);
foreach ($arr as $key => $value) {
    $arr[$key] = $value * 2;
}
// $arr is now array(2, 4, 6, 8)
```

至少我看到的代码下来，很少有人会用引用来修改数组的值，大多数都是采用的上面的方法

> ## foreach 的实现原理

感觉以上纯属自己扯着没事干……

所以查了官方的文档，并没有解决我的疑惑，因为我知道没用引用的时候`$value`只是个副本，用了引用就成了指针这玩意儿了，但是并没有说`$array`是什么情况，而且，从以上代码的表现来说，很有可能不只是`$value`是个副本，`$array`也是个副本，当使用引用了，那么两者没有再拷贝一份

所以下面我要干的事就是查看foreach的原理，也就是源代码，看看究竟怎么处理的，然后证明自己的猜想。

好在鸟哥有一篇博客写了![关于foreach的原理](http://www.laruence.com/2008/11/20/630.html)

按照鸟哥的这篇博客的思路，foreach实际上是被这么解析的

``` C
T_FOREACH '(' variable T_AS   { zend_do_foreach_begin('foreach', '(', $arr, 'as', 1 TSRMLS_CC); } foreach_variable  foreach_optional_arg(T_DOUBLE_ARROW  foreach_variable)   ')'
{ zend_do_foreach_cont('foreach', '(', 'as', $key, $val TSRMLS_CC); }
//循环体内语句
{zend_do_foreach_end('foreach', 'as');}')}}')'
```

所以不难看出，foreach最重要的就是`zend_do_foreach_begin`,`zend_do_foreach_cont`, `zend_do_foreach_end`这个三个函数，具体这三个函数是干啥的，还是看鸟哥的博客或者自己看源代码吧，不是这篇文章的重点

http://www.dataguru.cn/article-9116-1.html

http://www.laruence.com/2008/11/20/630.html
