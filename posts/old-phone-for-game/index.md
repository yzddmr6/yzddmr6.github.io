# 旧手机改造游戏机


<meta name="referrer" content="no-referrer" />



## 背景
最近安卓yuzu模拟器突飞猛进，能玩的游戏越来越多，画质也越来越好。
家里正好有一个吃灰的小米10，865，8+128，属于yuzu跟开源驱动优化的还比较好的型号。就准备把它改造成一个游戏机，发挥一下余热。
## 刷系统
MIUI是不可能MIUI的，太臃肿了。最开始小米10刷的EvolutionX，但是更新了最后一个版本之后掉电嘎嘎快。待机的情况下一天掉20的电，压都压不住，作为游戏机肯定是不行的，还没开始玩就没电了，需要重新换个系统。
找了一圈支持小米10的类原生，综合对比决定刷tequilaos.org这个ROM，不那么臃肿，也不像lineageOS那么毛坯。使用一段下来感觉不错，配合冻它待机一天也就掉4-5格电，电池健康度本来也不行了也可以理解。
这里还有一个插曲，发现tequilaos的内核不支持zstd等zram方式，就重新build了一个白狼内核刷进去，毕竟8G的内存比较紧张，zram肯定是要拉满的。
## Root与软件
作为游戏机，其他没用的能不装就不装，尽量追求精简。类原生压后台的能力都不太行，避免安装国产流氓3A大作。下面是我装的一些常用软件：

- Scene：必装软件，性能调度/功耗监控等。
- 爱玩机工具箱：很多功能很方便。
- MT管理器：安卓下无可替代的一款文件管理器，支持APK反编译等，功能强大。
- 冻它：支持图形化，日志也清晰能体现出电量变化，个人比较喜欢。
- konabess：GPU超频/降压调节软件。
- 酷安：主要功能是装软件以及抄作业。
- yuzu：开源Switch模拟器。
- Shooting Plus：手柄映射软件。
## 降压超频
865还是有点东西，超频到905mhz一点问题没有，但是电压就得上turbo了，turbo耗电实在太快了。
经过多次测试，把最大频率调为855，电压NOM_L1，这样功耗跟性能都比较均衡。
![Screenshot_20240121-152410.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1705821924953-3807f846-de54-4774-b926-6232f2cfa12c.png#averageHue=%232f2f2f&clientId=u40296d5f-8fb8-4&from=paste&height=1036&id=u8c16a2ad&originHeight=2340&originWidth=1080&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=118783&status=done&style=none&taskId=u5bb172ed-febb-4479-9c8a-384d73d7ef8&title=&width=478)
超频之后跑一下分还行，比原来好多了，原来是4k左右。
![1705808765165.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1705819849943-deb2ef94-9792-46a7-b7de-d59c171b67a9.png#averageHue=%23ebe8e5&clientId=u40296d5f-8fb8-4&from=paste&height=936&id=ud0f1cf20&originHeight=2340&originWidth=1080&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=832887&status=done&style=none&taskId=uf64b8ee8-8466-44b0-861a-04f26ba9431&title=&width=432)
## D8手柄键位调换
主要是当Switch模拟器来用，直接搓玻璃差点意思，还是要选一个手柄。综合对比下来D8算是性价比非常高的一款手柄，100出头有霍尔摇杆，支持自定义映射以及宏功能。
但是美中不足的是D8是Xbox手柄格式，abxy的位置跟Switch的是相反的，这就导致非常的别扭。另外yuzu的映射非常的奇怪，xy的实际功能是相反的：按下x实际上对应的是y，y对应的是x，不行，得改造一下。
![IMG_20240120_115235.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/1599908/1705819517932-1f10954e-e716-4f2e-8279-e32b10c1094d.jpeg#averageHue=%23dad9c5&clientId=u40296d5f-8fb8-4&from=paste&height=2458&id=u6b2aa623&originHeight=3072&originWidth=4096&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=3995063&status=done&style=none&taskId=u0f7f32e9-7c5d-4129-82be-3bd762c546f&title=&width=3276.8)
首先是调换abxy的位置，Xbox强改Switch键位，拆就完事了。
![mmexport1705725553859.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/1599908/1705819732615-60f8f88a-565a-4cdf-8f86-d34a3e797f18.jpeg#averageHue=%23afa992&clientId=u40296d5f-8fb8-4&from=paste&height=2429&id=u7020a461&originHeight=3036&originWidth=4042&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=5591740&status=done&style=none&taskId=u8d94ce1b-d647-416d-b681-9b4ae636e9a&title=&width=3233.6)
一番操作之后，好消息是键位换过来了，也能用；坏消息是怎么多了个零件出来？？？
![IMG_20240120_123148.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/1599908/1705819753307-8e20f829-81e8-477e-bf80-899eb4a32023.jpeg#averageHue=%23bebca7&clientId=u40296d5f-8fb8-4&from=paste&height=2458&id=ucc653d3e&originHeight=3072&originWidth=4096&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=4035288&status=done&style=none&taskId=u49dda5b8-70b6-4b69-8e55-e05ee9302b6&title=&width=3276.8)
## 逆向yuzu修改映射
换过来之后，看起来是舒服了，但是现在又变成ab键是相反的了：a->b，b->a。
安卓yuzu目前还没有按键映射功能，为了解决映射的问题，尝试了好几个方法：

1. yuzu+。B站上有大佬给yuzu添加上了按键映射的功能，但是很久没merge主线的更新。试了一下很多游戏还打不开/黑屏，放弃。
2. Shooting Plus。这是D8自带的一款映射软件，简单来说就是可以把手柄的按键模拟为屏幕的触摸，这样就可以用手柄玩王者荣耀原神这类原本不支持手柄的游戏。本来这是一个通用而且完美的方法，但是发现居然无法映射- +这两个按键。。。导致有些游戏里面打不开目录/设置等功能，遂放弃。
3. 逆向yuzu。yuzu代码开源，翻了下源码控制映射的代码在这里：[https://github.com/yuzu-emu/yuzu-android/blob/master/src/android/app/src/main/java/org/yuzu/yuzu_emu/utils/InputHandler.kt](https://github.com/yuzu-emu/yuzu-android/blob/master/src/android/app/src/main/java/org/yuzu/yuzu_emu/utils/InputHandler.kt)。其实可以修改源码后编译，但是里面有很多JNI库需要编译比较麻烦。反正也没加壳啥的，直接逆向修改后重新打包。过程可以直接看B站上大佬的：[https://www.bilibili.com/read/cv28837309/](https://www.bilibili.com/read/cv28837309/)，改成如下内容然后重新打包签名。这里还有一个问题，如果不卸载原来的yuzu会报签名不一致无法安装，lsposed安装核心破解绕过签名即可。

![1705808765192.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1705820789954-40c2cbec-3e6c-423d-8ba1-1bf5a1b7e4c4.png#averageHue=%23e0dfdf&clientId=u40296d5f-8fb8-4&from=paste&height=923&id=u15c275af&originHeight=2340&originWidth=1080&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=233185&status=done&style=none&taskId=u56cef516-d1d6-483a-8632-9ad73b9534e&title=&width=426)
## 效果
终于，一个比较理想的游戏机诞生了，玩玩马里奥或者塞尔达无双这类游戏还是没问题的，旧手机不用再换菜刀换剪子了。
![IMG_20240121_152023.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/1599908/1705821702470-202182c6-8075-4f1d-a492-e0007bc2cbcd.jpeg#averageHue=%23374d38&clientId=u40296d5f-8fb8-4&from=paste&height=2458&id=u0a04cac6&originHeight=3072&originWidth=4096&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=3930935&status=done&style=none&taskId=ua975733e-d0ac-4323-b73f-5031474f758&title=&width=3277)
![IMG_20240120_223317.jpg](https://cdn.nlark.com/yuque/0/2024/jpeg/1599908/1705808849736-51e0a4ef-dfcf-4186-b477-0201231d783f.jpeg#averageHue=%23959ca5&clientId=ua53e668b-1ed2-4&from=paste&height=536&id=DvtG5&originHeight=3072&originWidth=4096&originalType=binary&ratio=2&rotation=0&showTitle=false&size=4380560&status=done&style=none&taskId=u47231aba-5303-402a-8493-13cb7a9ac01&title=&width=714)
不过模拟器还是图一乐，实际游戏体验并不是非常好，经常会闪退黑屏，导致塞尔达无双这种中途不能存档的游戏打了半天白打了。改造主要是享受折腾的乐趣，有能力的话还是支持一下正版。
