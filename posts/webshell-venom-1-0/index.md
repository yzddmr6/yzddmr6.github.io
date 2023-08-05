# 利用随机异或无限免杀d盾


```
自己三个月前在圈子发的文章，保存一下
```

<meta name="referrer" content="no-referrer" />

## 前言

最近D盾更新了,在某司某圈也看到了不少免杀d盾免杀狗的一句话帖子

但是基本上只要放出来不到两天时间就加入查杀全家桶.最近一直在造各种车轮子

就想着其实可以写个脚本利用异或来fuzz出指定的字符

然后拼接出assert或者create_function等函数,来对抗waf的检测.

## 思路及实现

### 首先如何fuzz的问题

先讲一个离散数学中的概念叫可逆 ,异或的运算就是具有可逆性的.

具体什么意思呢,就是说若ab=c，则有bc=a

所以只要把需要拼凑出来的字符串a跟随机取出来的符号b异或,然后出来的结果c就是需要跟b异或的内容.

举个例子

我们来echo一下字符a跟符号*异或的结果

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431806-98acf76b-c8c6-40a7-a452-723a1531bf8c.png)

是大写字母K

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900431914-0c92f75c-7ed1-4313-a969-13ab7a910376.png)

然后把大写K跟*异或

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432004-f8ee8962-e6fa-420e-910f-24e04904924c.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432104-d48159b4-e2e8-420b-a52e-80420271ee76.png)

就出来了我们想要的a

那么也就是

```
a=K^*
```

但是在写的过程中问题来了

很多时候异或出来的字符是不可见的小方块

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432206-1d31f2ae-c013-4a13-af7b-700579478e24.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432320-9aa2787b-1c1b-46df-ae4b-13ce57b352eb.png)

就需要把它编码

看了以前有一篇文章是用url编码

但是在实现过程中发现url编码也有一定概率出现不可表示的字符

那就开开心心上hex吧

最终成功拼接出来了assert

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432411-94f6a97f-8f19-4c58-9d5f-9634e4716142.png)

接下来就是写个字符串池子,用来存特殊符号,然后随机取出来进行异或,拼接想要的字符.

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432512-3e9d503b-c6ee-413a-8281-343edb9253f3.png)

把它封装成函数

可以设置需要异或的字符串长度

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432632-a81a2d99-946c-4093-a5a6-ac93c4c4f729.png)

其实也可以用中文甚至emoji表情来异或,但是考虑到乱码还有不同系统对表情的支持不同,就算了.

取出拼接好的assert,把get的数据传进去,就成了下面这样

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432720-63fa07f2-fabb-410f-b57e-61a66eb98f04.png)

哈,看来可以使用啦

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432811-7a2caf42-9cdf-4c0c-a6a7-4a6652d84e0c.png)

接着是调用的问题,其实到了这个地步

不管你什么d盾安全狗已经认不出来函数里面写的什么意思了

但是他会根据函数的调用来检测拦截

如果这个时候直接调用的话会爆一级可疑函数.

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900432911-c0cb17dc-c658-4bb6-87a5-2812ad7196cb.png)

既然做免杀肯定要0级了啦

放到类里面再调用就好了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433015-03c03e1c-4fbc-414f-807f-fbfda2824dd4.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433125-b8a93093-4a09-463e-8760-f8806f77f9a2.png)

接下来就是造轮子了

在脚本中为了增大waf识别的难度 类名方法名也随机化了.

## 使用方法

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433239-1810b3a0-1794-41b5-b986-19e2a354cee2.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433354-70611595-af62-4b5a-b94c-0582dcb78a70.png)

右键查看生成的源码

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433442-24389ac2-132c-4f2f-97e5-25d3d3c315d0.png)

已经保存到同目录下1.php里面了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433572-1fea790c-9c48-408b-abea-b6ca930466aa.png)

生成了十几个

附上过D截图

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900433694-f736dfc3-09cd-48bf-8c84-51d90df23fdc.png)

## 最后

### 脚本特点有三个

一利用特殊符号异或达到迷惑waf的目的,并且因为每一次的拼接都是随机生成的,所以单单一个文件进了特征库也不用担心

二是利用类调用,类名函数名随机化,杀软分析起来可能跟普通的文件没有什么区别

三是没有assert eval create_function 等这些关键字,更为隐蔽.

因为是随手写的,所以代码比较糙,不过不要在意这些细节……

只是提供了一个思路,其实大马也可以像这样写个免杀模版,下一篇文章再讲吧

虽然都是随机化,也没有assert eval 这种关键字,但是用的人多了当然脚本的免杀性也失效了,可以关注我的github:

https://github.com/yzddmr6

以后更新的免杀脚本都会放在上面.

如果有什么讲的不对的地方还请大佬们多多包涵.

## 源码地址

https://github.com/yzddmr6/webshell-venom
