# 向吃鸡外挂站开炮（二）


<meta name="referrer" content="no-referrer" />


> 上一篇 [向吃鸡外挂站开炮（一）](https://yzddmr6.tk/posts/hack-pubg-cheat-site-1/)

## 前言

首先打开网站我们可以看到他的炫酷界面

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900398292-8dd73a1a-9b8b-4ea4-a68f-91a7b6502e58.png)

暖心公告

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900398774-a498a4ef-b5fb-41b2-a7f7-1faee531288b.png)

不要脸的宣传词

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900398935-b9a763ba-e7fc-4a22-b726-f5d81538c452.png)

## 发现注入

基于tp3开发,后台/admin

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399214-afc739eb-6149-4171-8cec-e13f3a94cd97.png)

尝试万能密码

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399313-fa18a3bb-cc23-4eed-b474-7abe9b82db8c.png)

提示密码错误

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399420-85a0cf71-9488-4244-8e7d-bc33057ee678.png)

尝试admin  admin888 提示账号不存在

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399510-5fd3230d-510a-4ed4-87ef-a412148659b5.png)

两者回显不同，考虑可能存在注入

## 无法利用？

burp抓包发送到repeater进行进一步测试

发现条件为真时返回`status: -2`，条件为假时返回`status: -1`

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399594-05c9e668-b481-4622-b714-a4fca773ac47.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399677-5921bebc-9191-4af0-aea1-18fbb7d96af8.png)

进一步印证了猜想，后台存在注入

扔到sqlmap跑

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399759-e0952986-0eb2-464f-a03e-48049d024c20.png)

无法检测出注入，提示一堆404 not found

开始以为是cdn封锁了sqlmap的流量，后来发现根本没什么防护。。。虚假的cdn

于是考虑可能是cms自身过滤了一些东西

## 绕过过滤

经过测试发现只要出现尖括号就会返回404

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900399930-697966ba-354e-4618-8ddd-d20f02678bc9.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400013-183195d3-e016-4dc7-94b3-e4eca3b0587e.png)

可以用between来绕过

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400106-09b80ae3-2872-4d29-a466-5f942315ad70.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400198-eb7456ef-6303-43cb-8e35-3c61bb731845.png)

这时就继续按照  条件真=>-2  条件假=>-1 来回显

也就满足了盲注的条件

忽然一想这个情景跟第五空间决赛的那道注入题一毛一样

真返回一个页面 假返回另一个页面  出现被过滤字符返回其它页面  并且要用between来绕过

CTF诚不欺我

所以只要在sqlmap的参数里加上`--tamper=between` 即可

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900400286-2d9cab5f-d7b3-49d3-8f7f-c40fd7d88699.png)

## 最后

数据库里管理员密码用的aes加密，没有秘钥，无法解密。

普通用户登录口被关闭，无法注册也无法登录。

除了脱出来一堆孤儿的信息其他也没什么用

打包一下证据，全部提交有关部门。
