# 向吃鸡外挂站开炮（一）


<meta name="referrer" content="no-referrer" />

## 前言

因为最近吃鸡被外挂打自闭了，所以准备也让那些卖挂的体会一下什么叫做自闭。

昨天晚上爬了快1000个卖吃鸡外挂的平台

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900394938-6732cd66-fbbe-45b1-9eb6-728f6e07b0f5.png)

你们这些卖挂的，等我有空了一个一个捶。

发现大多数都是用的一套aspx的程序，可惜没有源码不能白盒审计，黑盒也找不到什么洞

只能找找软柿子捏

昨天晚上一口气锤了四个

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395052-0b992318-fa59-42a8-b0e6-ab586d72f2ce.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395170-94b7ebf9-7581-4cba-a17e-86d6b74e24d1.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395410-78b88f9e-05e9-421d-8ef4-eaa3a1d31d66.png)

基本上都有宝塔

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395522-11fab4ee-8837-4c08-b980-1771e73704d8.png)

不过php-venom 4 系列加上配套的编码器过宝塔稳得一批

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395618-4eacd63d-121d-4439-a13b-ae981e936a10.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395744-c1b8958b-2623-4652-b55b-599905c37fae.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900395918-c727a9a3-2a4f-47c5-9bd4-69f27623c873.png)

脱了裤子发现里面4000+孤儿

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396002-c81c2c89-e58d-435f-b9f1-bf20070df004.png)

今天晚上又锤了一个吃鸡外挂站

可惜尴尬的是没有写入权限

写篇大水文记录一下

## 正文

### 毫无套路进后台

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396077-6cd624e4-a042-4fc3-a3f9-329c9d6a0973.png)

这个应该算是那种推广站，里面什么都没有，只有宣传内容

管你是什么，照锤不误。

看了一下是织梦二次开发的站

后台很容易进，这里大家都明白什么意思。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396189-be0255c0-1697-43cf-b14c-cf0027e5eead.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396316-91d56db8-5ed7-4434-a0b5-e03a55d6a0f9.png)

### 玄学后台

发现后台删了很多功能，特别是织梦的坑货文件管理器

但是从经验上来说很多这种二次开发的并不是真的把编辑器删掉了，只是在后台页面不显示了。

审查元素启动

随便找个链接改一下，替换成`media_main.php?dopost=filemanager`

然后点击，果然找到了文件管理器页面

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396433-c5b07886-debb-4192-b7e7-bcf17d2e1850.png)

上传shell

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396580-f4ea071c-8619-4647-9e2f-37684156c00e.png)

本来以为就这样结束了

结果发现虽然提示上传成功但是啥都没有

还以为是waf，就换了人畜无害的一张jpg上去也是啥都没有

以为是目录权限问题

找到session的临时文件，上传，照样不行

图就不放了，总之就是传不上去

觉得可能是整站没写权限

随手试试删除功能，发现可以删文件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396681-3e796c2f-1db5-4ebd-853a-43bf186ba8fc.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396782-bef7e4f7-6008-45c2-83cf-83ad71530632.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900396868-421cef70-4209-4012-acbb-6c7b63213f73.png)

emmmmm，所以到底是有权限没有呢

一般来说没写权限的话也就没有修改权限，也就是没有删除权限

想着是不是上传功能坏了，换个方法getshell吧

### 全员gg

首先想到的就是改文件，里面放个shell

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397061-c7f41698-a81d-4080-8b3b-f603c3e23e25.png)

显示csrf token不对

搜了搜怎么解决

发现是直接改check函数，第一句加上return

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397180-e487aa5d-91e5-4a49-a36c-e2c321b2e988.png)

结果修改config.php 文件也弹这个错误

所以就陷入死循环。。。

改标签也是一样的错误。

然后试了织梦的各个0day，后台代码任意执行

提示执行成功了，但是要么404页面，要么就是csrf token报错

为啥老是csrf token检测失败，以前就没遇到过这种问题。是我操作不对吗？

如果有表哥知道为什么的话麻烦告诉我一下谢谢

### 柳暗花明

本来想想算了，然后出去吃了个饭。

然后想着既然是弱口令会不会有其他人的后门呢。

就想起来织梦有个自带的后门查杀功能

同样的审查元素，找到后门查杀功能，开始扫描

果然发现可疑文件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397307-8efc5c9d-0bb0-4acb-ac9b-9fc5240dae91.png)

然后一看全是其他人的后门

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397420-72b8e411-943b-4d56-b6c3-c15ee5030b4a.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397520-2be01d9b-2027-4e1a-ac18-3c297451353c.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397623-0a661c27-7992-4ae1-956a-70ca66325558.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397735-59fe5634-dd90-47b3-9f12-5294a9cc678f.png)

随便找一个，连接上去

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900397901-af6de05e-b767-4d81-8583-5d2657ceae39.png)

getshell

## 最后

发现是星外，并且全站没有写入权限，难怪传不上去。。。

翻了翻目录，不能跨站，没写权限，无法bypass disable function

等于是啥都没有。。。

但是神奇的是可以任意文件删除

站就不删了，保存一下证据，提交网警。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900398005-09f5cb0b-bbc3-482d-af47-b260a55c1114.png)
