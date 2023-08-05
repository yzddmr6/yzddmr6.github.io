# 中国蚁剑源码分析


<meta name="referrer" content="no-referrer" />

## 前言

发现很多同学对于蚁剑的基本流程还有源码结构不太熟悉，所以就有了这一篇比较基础的文章，来讲一讲自己对于蚁剑的一些认识。

通过阅读本篇文章，你可以了解蚁剑的源码结构、运行流程、以及自己动手diy时要注意的几个地方。

## 正文

### 目录结构

```
/antData/	用户目录
/modules/	蚁剑后端模块
/node_modules/	安装的node模块
/source/	核心模块
	/base/ 	自定义的功能类
	/core/	payload模板
	/language/	语言模块
	/modules/	显示模块
	/ui/	UI模块
	/app.entry.js	渲染程序入口
	/load.entry.js	前端加载模块
/static/	静态资源文件
/views/		前端文件
```

其中最核心的是modules目录跟source目录。modules里的内容为蚁剑的后端模块，属于主进程。source中存放着蚁剑运行的核心代码，属于渲染进程。

### 如何debug

蚁剑的开发栈主要是：javascript / nodejs / electron。

Electron是由Github开发，用HTML，CSS和JavaScript来构建跨平台桌面应用程序的一个开源库。 Electron通过将Chromium和Node.js合并到同一个运行时环境中，并将其打包为Mac，Windows和Linux系统下的应用来实现这一目的。通过Node它提供了通常浏览器所不能提供的能力。

简单来说就是chrome里跑nodejs。

所以想要对蚁剑二次开发，要首先熟悉一下nodejs的基本语法。

electron里面又分主进程跟渲染进程，对于主进程的调试需要用到vscode等，而对于渲染进程只需要用到蚁剑中自带的dev tool就可以。具体可以看这篇文章：https://blog.csdn.net/gary_yan/article/details/78973336

一般来说，我们并不需要对modules中的后端模块进行修改，所以一般不会用到主进程调试，仅仅蚁剑中自带的dev tool就可以完成我们日常的调试工作。

打开蚁剑->调试->开发者工具即可看到调试工具。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379201-4772fcb7-ef45-40a9-923a-0e7f20bcc8b3.png)

是不是跟chrome一模一样？

其中console用于打印输出日志，蚁剑中默认的日志只会打印前100个字符，如果要查看完整日志需要输入antsword.logs[id]查看，在这里我们直接查看所有日志。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379387-87199788-06fb-4a87-9b52-e190f41a4851.png)

我们先连接上本地的shell，然后打印完整日志，就可以看到我们发包的很多参数，包括shell的配置，	编码器设置，字符编码，返回内容等等

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379496-2fa9a42f-86a0-4f5c-83ac-751baaf49761.png)

那么我们在哪里下断点呢

答案是在控制台sources->no domain下面,打开后我们可以看到渲染进程中加载到的各种资源、模块

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379637-5e0aa02d-b9a4-40d2-b6f1-782813864858.png)

然后我们找到想下断点的文件，就拿php的base64编码器为例，在其10行处点击一下会出现蓝标，就表示下断点成功。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379749-84f8a1df-495d-42e2-adf5-759ba0f4d1af.png)

此时我们在shell管理界面右键->刷新目录，就可以看到程序已经断到了我们下断点的地方，在右边可以看到此时的调用栈还有各种变量信息，就可以愉快的调试了。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900379979-35bb1753-454e-42d6-9c7a-47d849f2c264.png)

### 执行流程

- 主程序入口：app.js
- /source/load.entry.js 前端加载模块

- /source/app.entry.js 渲染程序入口
- /source/modules/filemanager/index.js 监听用户操作

- /source/core/php/template/ 提取组合Payload
- /source/core/base.js  发送事件与配置到后端request模块

- 解析、回显

就按刚才php base64编码器为例，我们看一下蚁剑是如何运行到这一步的。

查看上一个调用栈，发现是进入到了编码器处理部分，编码器会接收到三个参数：shell密码、初步payload、还有ext参数。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380144-62b4e2de-d426-4c52-b740-a2cb616ae23f.png)

其中ext参数即为shell的配置信息还有rsa对象的组合，这也是为什么我们在写编码器的时候可以直接获取到shell的各种配置信息。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380284-dddbe9cc-17b0-430b-978e-3ffdb0589071.png)

再往上看，发现complete函数调用了encodeComplete函数，complete负责将payload套入到模板中，并且设置数据前后分割符，发送给encodeComplete进行处理。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380391-084335b8-1ddd-4c88-b994-721354ba1d36.png)

再进入到core/base中的request函数，此函数负责将组合完成的数据包发送到后端的request模块。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380522-0c24191e-f1e6-4ff0-b540-205011daeccc.png)

那么是如何触发到这个请求功能的呢，我们直接跳到最开始的点击事件来看。

发现是当我们点击刷新目录后，会触发refreshPath函数。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380651-6a0cf8c5-6321-43f4-9136-8a54526835b3.png)

然后refreshPath函数分析是否有传递的路径参数，如果没有则为刷新当前目录。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380772-2218e200-4547-4d53-b374-6af27ea9b492.png)

然后gotoPath调用了this.manager.getFiles函数。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900380925-2547512e-a03d-4aa0-8aca-ec5e5742f8a6.png)

getFiles函数调用this.core.request，第一个参数为this.core.filemanager.dir，即为payload模板中的dir部分。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900381258-ffef63b4-268d-42e7-8e5c-fa4877c4a7b0.png)

接着组合、发送payload数据包，获取回显并解析。

其中解析跟回显部分不是我们关注的重点，我们关注的重点主要是提取组合payload到发送最终数据包的阶段。大家自己调试一下就明白其中的流程了。

### 修改数据包的几个重点位置

其中，如果要修改发送的数据包，有三个位置可以供我们参考。

- \source\core\base.js#187	模板组合（作用对象为全体）
- \source\core\php\encoder\base64.js	编码器处理（作用于当前类型）

- \modules\request.js	最终发包（不建议修改）

不建议修改后端最终发包是因为蚁剑中后端默认不能获取到所有的opt配置内容，除非自己加，我觉得比较麻烦。

### 配合opt参数实现自定义设置

opt参数中有shell的所有配置，通过此项可以做到动态修改数据包的内容。比如说我在[基于随机Cookie的蚁剑动态秘钥编码器](https://yzddmr6.tk/posts/antsword-xor-encoder-2/)中就是利用`ext.opts.httpConf.headers['Cookie'] = xxx`在数据包头部添加了一个cookie作为秘钥

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900381368-4e0fbd8b-5db3-413e-90bf-71ad86ef3f41.png)

在编码器中要用`ext.opts.xxx`来访问你想要访问的配置内容，在其他地方一般用`this.__opts__.xxx`或者`opts['xxx']`即可。

## 最后

在实现蚁剑jsp一句话的过程中，我使用了额外传递参数的方式来决定采用什么编码器、什么字符编码等。大家可以看一下我在编码器中的写法。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900381513-32361038-1854-48cf-b4cc-6b8a931e7b88.png)

这个方法是挺简单，但是特征也比较明显。那么怎么办呢？

相信你读完这篇文章后已经可以试着自己去改掉这个特征，有好的想法欢迎跟我交流。
