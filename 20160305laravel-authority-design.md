title: "基于laravel框架的后台权限设计"
date: 2016-03-05 13:58:20
tags:
- laravel
- PHP
categories:
- experience
---

以下只是我在毕设系统中设计的权限管理的一套方案，算是第一次造轮子写这个，不成熟的地方还请指教（0.0）

> ## 扯下权限控制

按照我的经验来说，权限控制的方式大体有两种

>- 通过前台的隐藏来控制用户可以进行哪些操作
>- 后台通过数据库中管理员分配的行为权限，查看该用户的角色和请求的行为是否合法

说人话就是，通过前台和后台两种方式控制。

如果只是前台控制，只要稍微懂点网页的，只需知道发送的数据，简单点可以通过url打开本来无权限的页面，高级点自己伪造post请求。

如果只是后台控制，极端点，用户点的功能然后一直返回说自己没权限访问，用户体验非常不好。

所以说了这么多废话，就是想说现在只要是正常的管理系统，都会有结合以上两种进行控制，毕竟面向的用户鱼龙混杂，林子大了啥鸟都有。。。不过这好像也是废话

> ##大体的设计方案

现在我讨论的只是后台的控制。

>- 同学A：按照每个模块增删改查，然后其他特殊功能再加上。
>- 我评价：控制粒度有点粗，不过这样的话前台对应好写，然而这样注定要写很多判断代码，作为写后台的我表示我很懒！！
<br/>
>- 我：我认为在mvc中，用户的每一个操作都对应与control层的一个函数，所以只要在调用control层的某个函数前看看用户有没有调用这个函数的权限就行。
>- 我评价：只要写一个通用的判断函数就可以搞定，代码量少，然而控制的太细，管理员在分配权限的时候会有点痛苦，然而代码少我喜欢！

由于后台是我负责的，所以理所当然的用我的方案，只是这样的话前台不好控制目前在考虑将按钮，链接什么的做成函数的形式,后台填写，这样的话前台也能控制。自己挖的坑，哭着跪着也要填完。。

> ##详细设计

现在大体方案有了，就先设计数据库，关键是用户的权限如何设计存储。由于毕设系统的角色是固定的，就只有老师、学生、管理员。而且目前来看，系统中有些功能对于目前角色的关联性还是非常强的，所以暂时不存在角色间权限的继承什么的。

参考了大大们的意见，大致考虑了三套方案。

> 以control中的一个类为记录的最小粒度，也就是一个control类就是一条记录，然后该类的方法权限以及角色的权限，则按json格式存储。然后当用户登录，就将他的所有权限写入到session中。

由于每次调用函数只要查找session就行了，效率会在一定程度高点，但是由于存储在session，对于一些实时性的功能支持性不高。

举个栗子，老师说，12点正式开始抢选课，然后肯定有很多人在11点五十几分就登录系统，然后12点老师在权限里面开启了选题功能，然而那些先登录的人看到12点过了还是无法选课就一脸懵逼以为系统坏了，等他重新登录，选题都被抢光了，然后痛骂写这个系统的人= =。说实话我不想背锅也得背。

其实以上问题也很好解决，增加管理员可以不通过权限管理来开启功能，再添加一个时间控制的限定。不过这些功能我还没写呢，有了这功能，我估计我的权限管理会采取这个方案，目前还不适合。

以下为数据库表的结构。

``` sql
+------------+------------------+--------+-------+-----------+----------------+
| Field      | Type             | Null   | Key   |   Default | Extra          |
|------------+------------------+--------+-------+-----------+----------------|
| id         | int(10) unsigned | NO     | PRI   |    <null> | auto_increment |
| model      | varchar(255)     | NO     |       |    <null> |                |
| name       | varchar(255)     | NO     |       |    <null> |                |
| authority  | text             | NO     |       |    <null> |                |
| created_at | timestamp        | YES    |       |    <null> |                |
| updated_at | timestamp        | YES    |       |    <null> |                |
+------------+------------------+--------+-------+-----------+----------------+
```

> 以control中的类的每个函数为记录的最小粒度，也就是control中的一个函数一条记录（不包括魔术方法），权限则是按照约定好的角色顺序弄成01字符串存储。而且每次调用函数都会查找数据库。

这是我目前在用的方案，方便查找，但是也就这个系统适用，当多个角色之间有继承关系或是新增角色的话就不适用。目前来讲还是可行的。

以下为数据库的表的结构。

``` sql
+------------+------------------+--------+-------+-----------+----------------+
| Field      | Type             | Null   | Key   |   Default | Extra          |
|------------+------------------+--------+-------+-----------+----------------|
| id         | int(10) unsigned | NO     | PRI   |    <null> | auto_increment |
| model      | varchar(255)     | NO     |       |    <null> |                |
| method     | varchar(255)     | NO     |       |    <null> |                |
| name       | varchar(255)     | NO     |       |    <null> |                |
| authority  | varchar(100)     | NO     |       |       000 |                |
| created_at | timestamp        | YES    |       |    <null> |                |
| updated_at | timestamp        | YES    |       |    <null> |                |
+------------+------------------+--------+-------+-----------+----------------+
```

> 按角色的权限为记录的最小粒度，说白了就是上一种方案中，将01串拉开而已。

这方案刚想起就被我弊了，可以说这个方案很适合角色不固定的情况，然而这个系统角色固定，所以采用这种方案势必会产生很大的数据冗余。

> ##关于laravel中的middleware

上面说了一堆设计的思路，下面就开始着手写代码了。具体在哪里控制，这又是一个问题。其实，在哪加的思路很简单，就是在route查找之后，进入函数之前。

在这之前咱们扯扯middleware，其实我觉得middleware就是为了权限而设计的，以下为刚初始化的middleware的类。

``` php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Contracts\Auth\Guard;

class AuthorityMiddleware {

    public function handle($request, Closure $next) {
        return $next($request);
    }
}
```

然而我发现一个事实，$request 中只有请求的url，并没有通过route查找传来的类名和方法名= =。要获得这些还得看框架源码，在一处地方开个默认参数传方法和模块进来（没试过）。

虽然同学B说直接按照url来控制，然而我是拒绝的。首先不说url与control的函数之间的关系不是一一映射，有的url只是单纯的返回一个静态页面，而且url里面有的含有参数，要正则匹配嘛。。。我是拒绝的！

通过以上思考，基本确定，我是要改框架的。感觉这个举动有点冒险啊，毕竟一个改不好，系统就容易炸=。=

> ##具体实现

上面废话了这么多，开始上代码吧。

在`vender/laravel/framework/src/Illuminate/Routing/Controller.php`里面有一个函数

``` php
    /**
     * Execute an action on the controller.
     *
     * @param  string  $method
     * @param  array   $parameters
     * @return \Symfony\Component\HttpFoundation\Response
     */
    public function callAction($method, $parameters)
    {
        return call_user_func_array([$this, $method], $parameters);
    }
```

没错，就是这个函数执行controller里面的函数，所以只要在这之前加个权限判断就行了。

改后代码

``` php
    public function callAction($method, $parameters)
    {
        if (!$this->getMethodAuthority($method)) {
            return view('noAuthority'); //返回没有权限的页面
        } else {
            return call_user_func_array([$this, $method], $parameters);
        }
    }

    private function getMethodAuthority($method) {
        $nameSet = explode('\\',get_class($this)); //传入的method中有namespace，获取最后的controller名
        if (in_array('Controllers', $nameSet)) {  //确认是controller中的类
            $controlName = end($nameSet);

            //查找数据库获取权限
            $resultSet = DB::select('select `authority` from `authority` where `model` = :model and `method` = :method limit 1', ['model' => $controlName, 'method' => $method]);
            if (count($resultSet) == 0) {
                //数据库中没有该函数的记录
                dd("no such record in sqlite! please add. model name: " . $controlName . "; method name: " . $method);
            }
            $result = $resultSet[0]->authority;

            //权限判断
        }
        return true;
    }
```

这就是通用的权限控制，只要数据库中有相应的记录，就可以控制，写的新方法也不需要为了权限控制改代码。

> ## 写在最后

其实，这段代码在我的系统运行时有一个bug的（跑。。。）。

一般系统会把用户的角色写入session中，啥时候写入呢，就是在登录的时候。没错，bug就是这个登录（=。=）。

我是通过获取session中用户角色来获取真正的权限，然后判断是否可以进去，然而登录时session里压根就没有角色，自然会报错。

解决方法有两个。

>- 当session中存入角色时，不进行权限判断。
>- 不从session中获取角色，而是查找数据库获取。

果断第一种啊。

The End~
