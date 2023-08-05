# 蚁剑改造计划之实现其他参数的随机化


<meta name="referrer" content="no-referrer" />

## 前言

本人有意写一份系列文章，主要内容是分享蚁剑改造过程中的一些技巧与经验。

因为蚁剑的相关文档实在比较少，可能很多同学都像自己当初一样想要二次开发可是不知如何下手。

不敢贸然称之为教程，只是把改造的过程发出来供大家借鉴，希望其他同学能够少走弯路。

- [蚁剑改造计划之实现动态秘钥编码器解码器](https://xz.aliyun.com/t/6571)
- [蚁剑改造计划之基于随机Cookie的动态秘钥编码器](https://xz.aliyun.com/t/6917)

- [蚁剑改造计划之增加垃圾数据](https://xz.aliyun.com/t/7126)

## 正文

### 历史遗留问题

我在前面几篇文章提到过，蚁剑一直有一个硬伤就是它对于其他参数的处理仅仅是一层base64。这就导致了不管怎么对主payload加密，WAF只要分析到其他的参数就能知道你在做什么。

例如你在执行cmd的时候，就一定会发送一个经过base64编码的cmd字符串，这就留下了一个被WAF识别的特征。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900373995-09645b59-3fd1-4998-a4e0-efdcd03cfddd.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374255-664ccb55-dea1-4d9e-80cc-48d2eaa71c36.png)

即使是蚁剑编码器仓库中的aes编码器也只是对主payload加了密，防护方在不需要解密主payload的情况下只要看到其他参数传的什么内容就能推测攻击者的行为。

yan表哥曾经在公众号中的[WAF拦了蚁剑发送的其它参数时怎么操作](https://mp.weixin.qq.com/s/ai3dW8H_ZnlFMPo-pgoqZw)文章中给出了一种解决方案。主要思想就是在不修改主payload的情况下，配合客户端额外再把它加密解密一遍。

可以是可以，但是很麻烦，对于普通的shell不具有适用性。

这篇文章的目的就是解决掉这个历史遗留问题。

### 随机化方式的选择

想要从根本上解决问题就要修改核心payload，那么怎么改呢？

以前师傅们的文章提出过两个方法，一种是把其他参数base64两次，还有一种是在其他参数前面加两个随机字符，然后主payload中再把它给substr截掉，来打乱base64的解码。

如果方法是写死的话，无非只是WAF增加两条规则而已。蚁剑这么有名的项目，一定是防火墙商眼中紧盯的目标。最好的解决办法就是加入一个用户可控的参数，能够让用户自定义修改。这样才有可能最大程度的逃过WAF的流量查杀。

所以本文采用的方法就是在每个第三方参数前，加入用户自定义长度的随机字符串，来打乱base64的解码。

这时，如果WAF不能获得主payload中用户预定义的偏移量，也就无法对其他参数进行解密。此时我们的强加密型编码器才能真正起到作用。

### 具体实现

思路

```
获取用户预定义前缀偏移量->修改核心payload模版->给其他参数前增加随机字符串
```

前端的话首先写一个text框，来获取用户的输入

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374380-93a91938-c86e-4665-a563-1b7fb578d46e.png)

在`\source\core\base.js`中定义randomPrefix变量

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374581-26dd0ddb-0a32-4657-b406-3c7c315ba352.png)

在`\source\modules\settings\adefault.js`中设置默认值

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374704-0091de9a-5183-4cf5-b87d-79ac9497522a.png)

然后后端就可以通过`opts.otherConf["random-Prefix"]`来获取用户定义的随机前缀的长度值。

修改模版前要简单了解一下蚁剑对于参数的处理流程

在各类型shell的模版文件中，会定义默认的payload以及他们所需要的参数，还有对于参数的编码方式。

```
source\core\php\template\filemanager.js
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374810-81263964-0ca4-4d9d-abda-d07217e2c53f.png)

在获取到模版之后，parseTemplate会对其中的参数进行提取、解析、组合，形成要发送的payload

```
source\core\base.js
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900374956-605a2fe5-9041-4a62-863c-afd37998d839.png)

所以我们要把用户预定义的前缀偏移量传入到两个地方：

（1）核心payload模版

（2）其他参数的组合模块

在核心payload中，我们将要修改的偏移量用`#randomPrefix#`进行标记，到parseTemplate函数组合最终数据包的时候将其替换。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375126-c9b2c1c4-e533-4d45-9b53-6c7dd31925f4.png)

然后定义一个新类型的编码处理器`newbase64`，在模板中修改对于参数的处理函数。

```
/**
       * 增加随机前缀的base64编码
       * @param  {String} str 字符串
       * @return {String}     编码后的字符串
       */
      newbase64(str) {
        let randomString=(length)=>{
          let chars='0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
          let result = '';
          for (let i = length; i > 0; --i) result += chars[Math.floor(Math.random() * chars.length)];
          return result;
      }
        return randomString(randomPrefix)+Buffer.from(iconv.encode(Buffer.from(str), encode)).toString('base64');
      }
```

修改后的模板长这个样

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375220-79dd6cc2-ad68-4419-b11e-025c2723f61d.png)

期间遇到一个小坑，就是无法在format()函数中获取opts的值

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375484-16e159f2-710e-436e-a8b1-55e733fbf500.png)

后来发现蚁剑中是这样写的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375620-06bf0dd8-1889-4c00-a81d-41ccbc9b5811.png)

还特意把原来的`new this.format`给注释掉换成`Base.prototype.format`的形式，具体原因我也不知道为什么。如果有知道的师傅麻烦告诉我一下。

既然追求刺激，那就贯彻到底，直接把opts传给format函数，然后在format中重新取所需要的变量。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375710-dab66dcd-5e68-4a09-bcca-8f27e19366e3.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375796-050475a2-87ee-44b7-9abb-3f5c4167cb8b.png)

### 测试

前缀长度默认为2，可以自行修改，只要不是4的倍数即可(原因自己思考一下)。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375894-4669a6d5-ee50-4b27-8486-ac0f63f28396.png)

可以正常使用

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900375982-10096851-a83f-4f2e-bb02-90e33216c9bf.png)

其中`prototype`为我们传入的第三方参数的值，在这里是要打开目录的绝对路径

```
prototype=ojRDovcGhwU3R1ZHkvUEhQVHV0b3JpYWwvV1dXL3BocE15QWRtaW4v
```

直接base64解码会是乱码

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900376144-3f30dac2-f077-4a5b-9c37-7d2dde3caaeb.png)

去掉前两位后我们进行解码则可以得到正确的结果。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900376229-141f6b38-d1fc-4a7f-94d8-7f68ffa4cfcf.png)

## 最后

偏移两位的效果可能还不是很明显，容易被猜出。但是当前缀长度达到10位以上的时候，就很难分析出最后的结果。

对php类型修改后我在本地测试了主要的13个功能，均可以正常使用。但是由于涉及到修改核心payload，等确定没有bug了再改其他的。

由于我是在父类Base中修改的编码模块，想修改其他类型的shell只需要照葫芦画瓢改一下对应的模版即可。

修改后的项目地址：

https://github.com/yzddmr6/antSword/tree/v2.1.x
