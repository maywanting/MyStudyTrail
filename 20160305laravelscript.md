title: "laravel框架中关于权限的一个脚本"
date: 2016-03-05 21:40:21
tags:
- laravel
- script
categories:
- experience
---

紧接上条博客。身为一个懒人，在开发阶段每写一个新的control方法就要在数据库添加数据，我是很不乐意的，于是就写了一个自动添加数据的代码，额。。。也称不上自动，毕竟还是要点按钮触发=。=

> ## 大体设计

上条博客说了数据库的表的详细设计。也就是说，目前数据库中需要的数据就是所有controller中的类，以及所有类的类名。

所以现在问题就是，如何获取controller中的所有类，以下简称C层类，如何获取C层类中的所有方法，还要除去魔术方法。

> ## 获取所有C层类名

在laravel中，C层类都有固定的格式，一个类一个文件，而且文件都是类名＋controller的格式。其实大多数的mvc结构的框架也差不多都是这样。

于是很好办了，获取controller目录下的所有文件名就行。

由于我这段代码是写在Repository中的，所以获取C层类的代码就是如下。

``` php
    $path = dirname(dirname(__FILE__)) . "/Http/Controllers/"; //获取C层类路径
    $handler = opendir($path);
    while (($filename = readdir($handler)) !== false) {
        preg_match("/^([a-zA-Z]+Controller)\.php$/", $filename, $modelName); //正则匹配获取C层类名

        //获取类内所有函数的操作
    }
```

> ## 获取C层类内所有的函数

再来明确一下接下来我们要干啥。我们已经得到了类名，接下来就是获取类内的所有公有函数，不包括魔术方法，而且不包括父类继承来的方法。

于是我们很快就会想到一个函数，用来获取类内的所有公有方法，没错就是`get_class_methods(class name)`，我一开始也用的这个。

然而有一个问题，这个方法返回回来的方法名，包括父类的公有方法，所以所有的C层类的返回的方法都包含父类的方法，这显然是我们不需要的。

如何解决呢，一开始我是想获取所有C层类的父类controller类的公共方法，然后做个集合的差，仔细想想工作量还挺大的。

后来在万能的stackoverflow上发现了[这个问题](http://stackoverflow.com/questions/12825187/get-all-public-methods-declared-in-the-class-not-inherited/12825317)，和我遇到一样的问题啊，解决方法就是用一个包装类ReflectionClass，解决的代码如下

``` php
$methods = [];

foreach ($relflection->getMethods(ReflectionMethod::IS_PUBLIC) as $method)
    if($method['class'] == $reflection->getName())
        $methods[] = $method['name'];
```

其实用这个类的返回的方法的结果和`get_class_methods(class name)`是一样的，一样包含父类的公有函数，然而比这个函数高级的地方就在于，它返回方法名的同时指明了这个函数在哪个类定义的，父类定义的则显示父类的类名，于是我们就可以用类名进行筛选，也算是解决。

ReflectionClass 类报告了一个类的非常多的详细信息，具体的可以看[官方文档](http://php.net/manual/zh/class.reflectionclass.php)，有非常丰富的功能。

> ## 具体代码

上面说了这么多，下面是综合的代码。

``` php
    public function refresh() {
        $path = dirname(dirname(__FILE__)) . "/Http/Controllers/"; //获取C层类的路径
        $handler = opendir($path);
        while (($filename = readdir($handler)) !== false) {
            preg_match("/^([a-zA-Z]+Controller)\.php$/", $filename, $modelName); //通过正则匹配获取C层类名

            if ($modelName) {
                $className = "App\\Http\\Controllers\\" . $modelName[1];  //加上namespace
                $class = new \ReflectionClass($className);
                $methods = $class->getMethods(\ReflectionMethod::IS_PUBLIC); //获取所有的方法名

                foreach ($methods as $method) {
                    if ($method->class == $className && preg_match("/^__[a-zA-Z]+$/", $method->name) == 0) { //确认方法的定义实在C层类中实现，而且不包含魔术方法
                        if (!$this->isAuthorityExist($modelName[1], $method->name)) {//数据库中有数据则略过
                            //不存在记录则增加,默认名字为方法名，权限为111
                            $newAuthority = new Authority();
                            $newAuthority->model = $modelName[1];
                            $newAuthority->method = $method->name;
                            $newAuthority->name = $method->name;
                            $newAuthority->authority = "111";
                            $newAuthority->save();
                        }
                    }
                }
            }
        }
        return true;
    }

    public function isAuthorityExist($modelName, $methodName) {
        $result = $this->model->where('model', '=', $modelName)
                        ->where('method', '=', $methodName)
                        ->first();
        return ($result) ? true : false;
    }
```

> ## 碎碎念

这段自动的脚本只能增加新增的方法，当删除某个方法时，数据库中还是无法删除。还有一个问题是无法同步子目录中的C层类，简单地说就是Controller里面还有一个目录，该目录里面的C层类是无法同步的，所以建议不要开子目录。

其实我写这篇博客主要是为了做下笔记，如何获取一个类的所有方法。我一开始写博客也是为了让自己记得某些东西写法，无奈于自己的记性实在不敢恭维。

The End~
