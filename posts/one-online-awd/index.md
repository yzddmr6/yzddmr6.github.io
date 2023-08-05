# 记一次AWD反杀之旅


<meta name="referrer" content="no-referrer" />

## 背景

星盟团队的内部awd训练，邀请我们团队参加，然后想着也没什么事情就报名了。

为什么说叫反杀之旅？

因为最开始的时候由于技术师傅的手误，前两轮设置后台check扣分的时候多按了一个0，导致一次check扣了我们1000分。。。

本来每个队初始分数是1000，结果我们一下变成了0，成了倒数第一。

不过经过团队的紧张的补救以及多年的手速，在多扣了几百分的情况下最后追赶到了正数第二名的位置。

觉得还挺有意思，所以今天写篇博客来记录下过程。

### 比赛规则

------

比赛时间

八月十四日（今晚）晚六点四十到十点四十

比赛规则

（队长比赛前五分钟找我私聊拿各自的登陆密码并尽快修改）

1.每个队伍分配到一个docker主机，给定ctf用户权限，通过制定的端口和密码进行连接；

2.每台docker主机上运行一个web服务或者其他的服务，需要选手保证其可用性，并尝试审计代码，攻击其他队伍；

3.选手可以通过使用漏洞获取其他队伍的服务器的权限，读取他人服务器上的flag并提交到平台上。

每次成功攻击可获得**5**分，被攻击者扣除5分；**有效攻击五分钟一轮**；

4.选手需要保证己方服务的可用性，每次服务不可用，扣除100分；**服务检测五分钟一轮**；

5.**不允许使用任何形式的DOS攻击**

------

因为是awd一般是线下赛，线上的awd确实还是第一次搞。

最开始的时候以为是在他搭建的内网里，还要端口转发到本地才能打，感觉麻烦很多。

后来才发现不是，就是一个外网主机开了很多docker，不同docker在不同的端口，打的时候IP都不用换，直接找端口就可以了。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900408155-2ed112e9-03d5-4387-8fa0-0ed8cfcb9856.png)

看了下界面还是挺炫酷的哈哈哈

## 冷静分析

拿到ssh后先登上去，改密码，保存源码，D盾扫一扫

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900408328-ee629092-4e69-4215-8d59-51da5291f27b.png)

虽然我经常发帖子过D盾，但是D盾确实很厉害啊哈哈哈，在比赛中还是非常有用的

老套路，被留了各种后门。

因为本来预定的环境有问题，启用了备用线路，所以只有一个web一个pwn，那就慢慢看吧。

### 第一个后门 任意命令执行

文件名 api.inc.php

很多文件都包含有这个文件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900408498-4c312cce-ba89-4404-a9c2-782ac9878eda.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900408670-b56b6c04-6d64-420b-bf15-4c8c17e0e500.png)

authcheckæ¯ä¸ªarray_mapï¼ä¸é¢51è¡åè°ç¨äºè¿ä¸ªauthcheck，

代码意思就是说收到数组authååbase64è§£ç ä¸éï¼ç¶åç¨authcode参数来任意命令执行

那么我们就可以构造以下payload：

```
http://xxxxx/api.inc.php?authcode=assert&auth=ZXZhbCgkX1BPU1RbJ2EnXSk7
```

auth参数解码后是

```
eval($_POST['a']);
```

就可以生成个用蚁剑连接的一句话啦

附一张当时的截图

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900408970-f45857f0-6b55-4159-88dc-4dbd54a1b94b.png)

最开始因为手速快，所有的人都被我们拿了flag

### 第二个后门  一句话木马

这个没什么说的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409124-5a001b91-cc1c-4955-bb6c-226b0ceeb974.png)

后来发现他的$islogin可以随便绕过。。根本不用登陆。。

### 第三个后门  LFI

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409262-7c8c5b77-0ed9-41bf-be09-89c089d7d28e.png)

直接包含flag就可以读取

### 第四个后门  任意文件上传

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409369-6885ef10-0054-4d98-bed8-e4c07b6ee301.png)

主要代码

```
if ($mod == "release"){
echo'<div class="panel panel-primary"><div class="panel-heading"><h3 class="panel-title">'.$xiaofu.' - 上传安装包</h3></div><div class="panel-body">';
if ($_POST["s"] == 1) {
$extension = explode(".",$_FILES["file"]["name"]);
if (($length = count($extension)) > 1) {
$ext = strtolower($extension[$length-1]);
}
if ($ext == "png" || $ext == "gif" || $ext == "jpg" || $ext == "jpeg" || $ext == "bmp") {$ext = "png";
}
copy($_FILES["file"]["tmp_name"],ROOT. "download/release/release.".$ext);
$city=get_ip_city($clientip);
$czip=($udata['dlip']);
$user=($udata['user']);
$DB->query("insert into `auth_log` (`user`,`type`,`date`,`city`,`czip`,`data`) values ('".$user."','上传安装','".$date."','".$city."','".$czip."','无')");
echo "成功上传文件!<br>（请刷新安装包文件夹）";
}
}
```

意思就是说，收到上传文件后检查后缀，然后保存。

但是仔细看这一句

```
if ($ext == "png" || $ext == "gif" || $ext == "jpg" || $ext == "jpeg" || $ext == "bmp") {$ext = "png";
}
```

如果后缀是这么多的一种，那么变量$ext 就被赋值为png

然后if语句就结束了

也就是说他这个判断后缀是没什么卵用的，任意的后缀就可以跳过他这个判断继续往下执行。

但是当时无法利用，因为网站权限被设置为不可写。。。

也就是说你只有ssh才可以更改网站的内容。

## 马失前蹄

因为漏洞都比较简单，很快就看完了。

简单说了一下分工，就让牛逼牛逼最牛逼巨佬naivekun师傅修洞，我们就开始写批量getflag脚本然后批量交flag

结果突然听见naivekun师傅大叫一声卧槽

看积分榜我们居然只有100分

其他队都是1000起步

喵喵喵？？？

我眼花了吗？？？

后来问kun师傅怎么修的，他说直接把参数删了。。。

**完 蛋，国 赛 重 现**

其实研究一下规则就知道，别人打你就扣5分

但是你修炸了就一下扣100。。。

唉，国赛就是这样被坑惨的，做题不少。。。但是越修分数越低。。。

但是也不会掉这么多啊，一下成了倒数第一

后来看群里才知道，由于技术师傅的手误，前两轮设置后台check扣分的时候多按了一个0，导致一次check扣了我们1000分。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409503-c2d1a876-b0af-45d6-a671-991647cc3d87.png)

然后我们手速比较快修的比较急，刚好在扣分的第一轮没修好就被扣分了

赶紧把洞重新修了一遍

gg，跟别的队差1000分还打个毛啊

猛男落泪.jpg

## 搅屎翻盘之旅

不过这个比赛有一点好处就是时间长，5分钟一轮，四个小时。

算了算我们还是有翻盘的机会的

毕竟手速快，顺便稳定了权限，基本上当时所有队伍的权限都在我们手里。

分工了一下

一个人稳定权限拿flag，一个人写批量脚本交flag，一个人搅屎。

pwn跟web是分开的，也差不多这个样子

当时自己批量getflag的截图

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409623-8bd7694d-4611-4091-a1cf-747e8372a520.png)

这是一个洞的脚本，多的图就没截了

然后慢慢分数就涨了上去，倒数第二还是第三吧

### 删库跑路

只靠得分不行，因为分差太小了，一轮全部拿到其他队伍的flag还没有扣得多

所以就想着**搅屎**

**因为别人扣分实际上就相当于自己队伍得分**

搅屎本来想删站的，结果没权限，就少了很多乐趣。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409744-d70a91c5-1424-4781-9bcd-57fb34169f36.png)

然后想着能不能fork炸弹啊磁盘写满啊这种

后来naivekun师傅发现可以删库让整个站500

哈哈哈哈快乐

删库跑路图：

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900409892-c437abdb-7c9a-4b7e-ba2a-634a027ffaea.png)

被群里人发现了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410021-de58c5ec-e0d6-493c-bf5e-c122c0282e71.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410127-ecf54265-26d1-44a4-b7ac-20bf39553158.png)

基本上把大多数队伍的库都删了个遍

但是我们后来发现

**删库居然不扣分？？？**

在群里问技术

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410244-512e1d9f-d378-4222-846a-694107d225ce.png)

原来check扣分异常。。。所以技术直接把check关了。。。

好的吧，反正继续搅屎没毛病

然后写个循环脚本，继续删库。。

**pwn也开始搅屎了**

附上队员blackbinary的犯罪证据

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410336-4e756e33-fa48-4830-b214-5bd9faebb2f4.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410495-bb7bb9c5-6187-491d-82d7-f3991573a094.png)

直接echo fuckyou到二进制文件里

这样其他队伍就无法攻击得到flag

## 逐渐回暖

后来因为搅屎比赛太过激烈

我们的队伍分数在逐渐往上涨，其他大多数被删库就在往下掉

最后就也不想打了，全部是自动化getflag，自动化提交了，大家也都自己干自己的事去了。

附上几张当时的截图

随机队名，我们是Colombia队

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410810-617291bf-2e8b-4b34-8a85-11da611ffe1f.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900410973-5a703587-0edc-4750-8a8d-ed28a296a487.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411073-e8a3015e-e393-4b09-97cd-c041ab1064d5.png)

第三了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411197-961af19f-64cf-4a3a-a062-1b9b0d465054.png)

然后第二了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411323-9c710b34-c596-4c52-9e39-c01316a57e79.png)

最后结果，第二名

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411468-d32a0375-0afa-4a8e-ac03-bf641ace4660.png)

## 后续

在多扣1000分的情况下从倒数第一反杀到第二

最后说给我们分数加上但是也没加，毕竟训练赛就当打着玩

还是挺好玩的，但是中途平台bug比较多

什么flag提交不上，提交频率过快

然而最让我懵逼的是

**他平台有漏洞**

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411643-f4d0aa06-49a2-4091-ba55-863d0db66c05.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411746-6e31b769-b4d4-4069-94b3-836be6118af1.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411834-c2fd0728-ebea-4eb1-9add-72fe26d99709.png)

每个队伍的cookie居然只是队伍名的md5。。。

阿伟死了

## 最后

总结一下反杀的原因还是**主要靠搅屎**，让其他队伍扣分

不然快1000分的分差靠**只有5分**一个的flag很难追上

不管怎么说这个线上awd还是挺有趣的

操作虽然都是常规操作，但也是对团队分工合作，搅屎技术的一次综合考验

特地写下这篇文章记录一下。

### 小插曲

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900411919-ea816a42-9d6e-4e32-b86f-4152b2dc14e2.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900412016-cd966e8b-8de8-4ed1-b080-28d93a10956a.png)

喵喵喵？？？

















