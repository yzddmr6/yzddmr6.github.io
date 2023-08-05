# 向吃鸡外挂站开炮（三）


<meta name="referrer" content="no-referrer" />

> [向吃鸡外挂站开炮（一）](https://yzddmr6.tk/posts/hack-pubg-cheat-site-1/)
> [向吃鸡外挂站开炮（二）](https://yzddmr6.tk/posts/hack-pubg-cheat-site-2/)

## 前言

第三篇终于写好了，也算是比较有意思的一篇，遇到了很多坑，也用上了很多小技巧，写出来博君一笑。

## 正文

毫无套路的炫酷界面

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400569-7eb25b32-5a41-4772-b78e-9d06991d610b.png)

琳琅满目的商品列表

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400752-7d93b70f-d820-4ba9-9556-99281841ed14.png)

这种卖黑号的通常都是跟各种hc商勾结在一起，用木马盗取用户账号，然后再出售账号让孤儿开挂。

### Getshell

getshell的过程懒得再复现一遍了，直接说思路吧。

发现用的是一套已知的发卡系统，Fi9coder刚好在我星球里发过此发卡系统的漏洞合集。

自己也审了一下，确实漏洞比较多。

随便找一处

submit.php里直接把$gid带入数据库查询，产生注入

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400912-92cc36d3-1c08-4c02-85ac-70365d29be0d.png)

然后到后台logo处未过滤直接可以上传shell

理论上很简单，但是实际上遇到了一些坑。

**第一个是对方似乎开了宝塔的防护，注入跟上传都会被拦截。**

我用的绕过办法是，注入用的参数污染，上传用的星球里嘉宾九世分享的bypass宝塔上传方法，换行绕过，一句话用的webshell-venom。

**第二个是找不到后台**

解决办法是找到他的一个旁站，然后谷歌找到了他旁站的后台。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401001-7eb22304-02b9-44fe-bae3-9674bd9b54a5.png)

国外服务器，旁站都是这种乱七八糟的非法站，用的都是同一套有漏洞的系统。

猜想可能都是这个后台，果然如此

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401082-ea2e1b67-935d-41e3-958c-a646f193e12e.png)

比较有意思，后台地址是 /sima ，看来站长也知道自己是个什么东西。

拿着注入出来的账号密码就进去了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401167-f1caed6e-5d97-4ee4-8766-9c05ce8038f4.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401269-84046678-6f94-438b-98ea-43d2b22a8bc8.png)

### Bypass disable function

确实是getshell了，但是连接的时候出了问题，蚁剑连接的时候一直转圈圈

发现是流量被检测后直接封ip了。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401368-b8cb7221-b759-43c8-863e-f7401ab1a650.png)

因为用的webshell-venom生成的马，兼容流量编码，所以开始解决办法是传一个id参数，然后用蚁剑的base64-bypass编码器，把所有的payload都base64一遍。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401448-a53edfe5-b309-4d27-962e-387810464085.png)

结果后面发现连接了一会儿又被封了。。。。

再换上用自己写的[蚁剑参数污染模块](https://yzddmr6.tk/posts/ant-mass-bypass/)，总算是不被封了。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401542-d91290c3-837c-46dc-9d56-1849cb638f77.png)

宝塔默认会禁用系统函数，需要Bypass disable function

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401639-02dce0d4-5052-4f2c-897f-75c642d13b96.png)

用蚁剑自带的bypass插件

发现因为宝塔的原因，会直接拦截上传的php

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401714-21dcae9a-6593-4691-a246-a21d2e3026a5.png)

解决办法是代理到burp后把payload抠出来，手动上传。

(我也不知道为什么自动上传不行手动就可以了。。。)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401792-6d74c550-9e03-4fbd-8d75-0251e79f7ed6.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401884-8914d6fb-6b5a-4837-b95f-afb58709e3af.png)

成功执行命令

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900401976-41803a1f-2c39-44ef-9103-bd4b405d1d67.png)

### 曲折的提权

发现是2015的内核后笑开了花，脏牛一把梭走起。

首先弹个msf回来，这样就可以有交互式shell

但是发现firefart的exp会直接把系统搞崩。。。服务器直接挂了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402056-af82f5b9-c644-4ac4-9928-ed089cfd274e.png)

本来以为是偶然情况，等到第二天服务器又恢复了又提了一次结果还是宕机。。。

所以又等了一天。。。

冷静分析一下，首先脏牛是肯定可以打的，因为内核版本确实在脏牛攻击范围内，并且firefart的exp确实有回显。

因为firefart是直接给你一个加了一个用户，并不会主动返回root权限的shell，需要你su切换或者去登录，猜想可能在覆盖的过程中造成了内核crash就宕机了。

猜想是否可以用直接返回root shell的exp来解决。

explit-db找到了这一个 https://www.exploit-db.com/exploits/40847

编译一下，加个-s参数保存/etc/passwd文件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402131-c65bf438-9a0c-41cf-bb8c-7763d53746b6.png)

看到了久违的root

### 拿下宝塔

先把他的passwd给恢复回去，不然一会儿机器又崩了就打不开了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402210-ea555cb4-8cd6-4c0a-adc4-72aede02abf1.png)

找到存有宝塔密码的数据库

```
/www/server/panel/data/default.db
```

同目录`admin_path.pl`下找到面板路径

扔到somd5里解密

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402297-a3692817-36c7-4f73-81d0-08b9571c9993.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402377-72d8de1a-2819-45c1-8f92-50b3350db676.png)

成功进入面板

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402455-8ea53eab-d392-4108-adff-320919dfc8bd.png)

其实进面板的时候怕他绑了微信有提醒，结果没有，哈哈

发现管理员登录IP

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402554-ddb85513-ca7d-49fa-9207-5e602acc8db8.png)

查一下地址是福建的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402642-16e63c67-30ef-4310-92ee-bd9a1f0c93cc.png)

拿到数据库root密码(虽然没啥用)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900402824-a3530978-30df-4726-bd6a-f173592377bb.png)

到此已经拿下服务器所有权限，清除痕迹擦屁股走人。

## 最后

有些图是后来补的，如果觉得有说的不对的地方欢迎跟我交流。

法律红线切不可碰，证据全部打包，提交给网警同志。

最后，吐槽一句吃鸡外挂越来越多了，越来越没游戏体验了。
