# bypass宝塔上传


<meta name="referrer" content="no-referrer" />

九世发在星球里的文章，因为比较敏感就放在博客上了。

绕过的方法：

三个点：

1.后缀名回车分割

2.删除Content-Type头

3.垃圾数据填充

1.后缀名回车分割

xxx.php

"xxx.

p

h

p"

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900389429-9e36af65-683a-40e4-bd83-b7dd83f71948.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900389543-d95754b2-63af-4916-9394-8e839c733546.png)

执行shell的时候，发现phpinfo()函数被拦了

当时第一个念头是：

草

第二个念头是

这个宝塔杀的肯定是敏感函数，比如执行:http://xxx.com/x.php?a=phpinfo(); 杀 phphaqinfo(); 不杀

那么只需要把haq置空就好了

最后得到新的shell

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900389645-b979ea6a-a1f7-43dc-8330-586c97a1211e.png)

成功过塔

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900389726-2f013c5b-ccdd-44e4-83d4-ea32d888a73c.png)
