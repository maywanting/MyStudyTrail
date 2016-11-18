title: "ubuntu下设置动态壁纸"
date: 2016-11-18 12:00:00
tags:
- ubuntu
- life
categories:
- experience

> ## 缘起

起因是这样的，自己实习的时候，由于是在无法忍受windows下开xshell然后全天在xshell下开发，于是装了双系统在linux下开发，然后挑壁纸就是一个必经的过程。我自己个人是比较喜欢26个字母系列壁纸的，排版好看而且纯色调，之前一直选的`F`字母，然后想换换口味，于是选择困难证犯了。

然后呢我就想搞动态切换，一分钟切换一次，这样就不用选择了。自己看了ubuntu下面的壁纸设计，确实有动态壁纸这一栏

![ubuntu下的默认动态壁纸](./picture/20161118_1.png)

下面有时钟标记的就是动态壁纸，所以弄动态壁纸是可行的啊

> ## xml文件配置

我的系统是ubuntu 16.04

ubuntu下的动态壁纸设置是在`/usr/share/backgrounds/`目录下。首先这个目录下存放所有可选的壁纸，可以将所有的动态候选壁纸存放在这。然后xml配置放在这个目录下的contest下。

![xml配置文件](./picture/20161118_2.png)

其中这个`xenial.xml`这个xml文件就是系统自带的动态壁纸的配置。另一个`may.xml`是之后我自己的动态壁纸配置。

> ## xml文件内容说明

它的配置其实很简单

``` xml
<background>
    <starttime> <!--这部分是这个动态壁纸啥时候开始用，可以设置为当前时间或者以前的时间-->
        <year>2016</year>
        <month>08</month>
        <day>18</day>
        <hour>00</hour>
        <minute>00</minute>
        <second>00</second>
    </starttime>
    <static> <!--这表示静态，不动的，单位为秒-->
        <duration>60</duration> <!--持续的时间-->
        <file>/usr/share/backgrounds/A.jpg</file> <!--壁纸的绝对路径-->
    </static>
    <transition> <!--这表示动态切换-->
        <duration>5</duration> <!--持续时间-->
        <from>/usr/share/backgrounds/A.jpg</from> <!--从哪张壁纸开始变化-->
        <to>/usr/share/backgrounds/B.jpg</to> <!--切换成哪张壁纸-->
    </transition>
</background>
```

以上简短例子的效果就是，壁纸A.jpg展示了60s，然后从A，jpg壁纸慢慢切换成B.jpg，持续时间为5s，然后立马壁纸A.jpg展示60s。

写好这个xml，然后保存，就可以在壁纸设置看到这个动态壁纸了。

![自定义壁纸配置](./picture/20161118_3.png)

然后就可以看到自己设置的动态壁纸，选中就可以了。

> ## 碎碎念

这玩意儿其实可以写一个脚本自己写个xml脚本，然而自己并不是那种频繁换批量壁纸的人，而且，这么简单的脚本肯定有人写了，哪天自己闲的蛋疼写个，目前自己忙成狗了就不写了。
