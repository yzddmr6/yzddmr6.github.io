# 绕过宝塔云锁注入钓鱼站


<meta name="referrer" content="no-referrer" />

## 前言

某天收到一封邮件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390006-ef2b22cd-8232-418a-8a58-77659b45c662.png)

一看就是钓鱼邮件，并且我也不玩LOL。

看了看感觉这个系统好像见过很多次，研究了一下，顺手日了下来

过程比较有意思，遇到了不少坑，写篇文章记录一下。

## 正文

### 信息搜集

打开网站首先我们可以看到他的炫酷界面

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390105-e4ed5146-dc70-4232-a883-fe4694d27a0b.png)

进一步搜集信息发现有宝塔+云锁，找不到后台，旁站全是这种钓鱼站，均使用了冒充官网的子域名前缀

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390354-824f80da-184d-4473-88a9-886876132905.png)

手工试了下发现有注入，但是有云锁

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390457-697bf907-deec-4d8e-972c-a51f808a68fe.png)

### 万能Bypass

还是利用星球里提过的方法来bypass

构造好post包后用sqlmap跑，发现有布尔盲注

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390559-f9cd619f-82eb-4fe8-8398-c0c9be0efabd.png)

本来以为就要完事了，结果sqlmap最后提示注入失败

emmmmmm，看一下发现被封了IP

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390667-fdc98995-0382-4ecb-a930-827dc7a6563b.png)

换个IP后，增大delay的数值，想了想他有可能是根据XFF来判断来源IP的，就又加了个`tamper=xforwardedfor.py`

哈，本来可高兴了，以为完事了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390750-45009393-ef1a-4193-b66c-ab9ac23df9a0.png)

结果发现跑不出来数据

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900390917-f1ef36f1-2387-4b17-8ab3-52fd85ec16eb.png)

exm???

### 遇到的坑

在从确认有注入到真正能跑出来的期间遇到了好多坑。。。

花了一下午时间才一个一个解决

#### 第一个坑：sqlmap的payload无法加载

抓包看一下，发现sqlmap的payload无法被加载到数据包里

相当于一直发送的都是没有payload的数据包，所以肯定注不出来。

具体原因不知道为什么，但是可以做一个猜想：

可能是构造的垃圾数据过多，文件过大，导致sqlmap还没来得及替换payload数据包就发出去了

解决办法就是减小数据包长度，然后抓包调整

最后发现30kb是个界限，刚好是sqlmap能发出去包，并且云锁跟宝塔不会拦截。

#### 第二个坑：win下网络阻塞

强制关闭sqlmap了几次，然后就发现网络阻塞，数据包在win环境下发不出去

解决办法就是换kali。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900391073-742476a1-8ce5-43ca-b814-23ed34602837.png)

#### 第三个坑：

sqlmap提示发现有无法识别的字符，解决办法是采用`--hex`

### 柳暗花明

解决完上面的坑后，终于可以出数据了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900391189-d7c8d813-bb97-420b-bf81-7ad2504c67ef.png)

跑了漫长的一个小时。。。终于跑出来了当前的用户名。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900391345-51c13a2c-8fb5-455c-917a-1137f45b6464.png)

然后是跑表名

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900391477-0f69da1f-73ea-4d32-a5f8-187c3859c8c5.png)

因为跑起来实在太慢了，后面就懒得跑了。

## 最后

面对这种邮件大家要提高警惕，一定要检查发件人跟域名是否是官方。

一旦遇到钓鱼邮件立马举报，防止更多的人上当。
