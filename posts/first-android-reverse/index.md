# 安卓逆向初体验



<meta name="referrer" content="no-referrer" />

## 背景
最近在捣鼓安卓yuzu模拟器，发现一个问题：JoyCon通过蓝牙连接手机后，两个手柄无法识别为一个设备。电脑上可以用betterjoy解决，安卓上搜了一下据说Joy-Con Enabler这个app可以解决。下载下来发现需要付费才能开启，就研究了一下安卓逆向。
![-1512082603.jpg](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696755238304-f975d37c-6d89-4904-9ad3-3e6a42217217.jpeg#averageHue=%23747c6a&from=url&height=1107&id=SNSsn&originHeight=2400&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=402602&status=done&style=none&title=&width=498)
## 过程
打开app，提示需要升级到PRO版，搜一下关键字

![-1543368485.jpeg](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696755243924-c6af6163-db9f-4d58-a4ea-7f721f93a0c8.jpeg#averageHue=%2395aac3&from=url&height=407&id=tbVl6&originHeight=1024&originWidth=1365&originalType=binary&ratio=1&rotation=0&showTitle=false&size=153702&status=done&style=none&title=&width=542)
![-1543368519.jpeg](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696755245841-0136a3ac-089e-4c6f-a232-f09dbde25483.jpeg#averageHue=%23151917&from=url&height=458&id=vhUOp&originHeight=1024&originWidth=1365&originalType=binary&ratio=1&rotation=0&showTitle=false&size=189739&status=done&style=none&title=&width=610)

dump出apk，拖到jadx里定位到关键点，还好apk没有加壳，其实也就是看Java代码。可以看到第一个点是会判断hVar.f1655b是否为joycon_enabler_pro这个字符串
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693622616592-1d28a73d-358e-447d-932f-83686abc105b.png#averageHue=%23fbfaf9&clientId=u8d8a2cd9-e9e2-4&from=paste&height=511&id=u5b135fd7&originHeight=639&originWidth=1045&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=66700&status=done&style=none&taskId=ue6b0bd6c-9bd8-4349-ab79-8f87b56c61f&title=&width=836)
安卓逆向不能直接修改Java，需要修改smali字节码，随便搜一个语法教程：[https://zhuanlan.zhihu.com/p/580962131](https://zhuanlan.zhihu.com/p/580962131)，把if-eqz改成if-nez，反转一下逻辑即可绕过。
修改smali有很多种办法，mt管理器比较方便，但是需要买会员，所以后来选了个破解的np管理器，修改后自动签名，安装。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693622651481-2616adc9-7d10-4e72-b3f7-f8d77b6a2662.png#averageHue=%23fefdfb&clientId=u8d8a2cd9-e9e2-4&from=paste&height=546&id=u76463a81&originHeight=683&originWidth=1059&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=63236&status=done&style=none&taskId=ua0aedd6d-f2f7-485f-9ce0-e184e675d3e&title=&width=847.2)
装上后发现不行，提示License校验失败，继续搜索关键字
![](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696757426460-d27f0c07-1bf7-47e8-9148-7c818da4b0b5.jpeg?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0%2Finterlace%2C1#averageHue=%23b3c8e1&from=url&id=RQBTs&originHeight=1125&originWidth=1500&originalType=binary&ratio=2&rotation=0&showTitle=false&status=done&style=none&title=)
发现这里jadx报错了，有一个小坑，需要改一下jadx的配置：文件 - 首选项 - 反编译 里面的显示不一致代码 选中 保存退出
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693622744072-6dfeaf71-5cce-4bfe-8283-baaee7541183.png#averageHue=%23fefefd&clientId=u8d8a2cd9-e9e2-4&from=paste&height=602&id=ubc8d7265&originHeight=752&originWidth=1311&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=86556&status=done&style=none&taskId=u5e75dbf6-d003-403d-bb67-327b49b5324&title=&width=1048.8)
然后就可以正常反编译了
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693622856855-87afa8d0-2923-4c65-99b1-adb82d12f056.png#averageHue=%23fdfbf7&clientId=u8d8a2cd9-e9e2-4&from=paste&height=370&id=u9185041d&originHeight=463&originWidth=1564&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=43465&status=done&style=none&taskId=u67620d9f-c21e-4f1b-b5ff-9cff8e3006e&title=&width=1251.2)
发现有一个这样的逻辑：
```
public void a(Context context) {
    try {
        if (!context.getPackageManager().getPackageInfo(context.getPackageName(), 64).signatures[0].toCharsString().contentEquals(getResources().getString(R.string.sin))) {
            this.f502f = false;
        }
    } catch (PackageManager.NameNotFoundException unused) {
        this.f502f = false;
    }
    R = true;
}

```
问问GPT，发现是检验包签名，如果发现被篡改就把this.f502f设置为false
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693624344954-6858c66c-287e-4f00-8f78-8e89a7388976.png#averageHue=%23cff7d2&clientId=u8d8a2cd9-e9e2-4&from=paste&height=457&id=u22754d25&originHeight=571&originWidth=1537&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=98759&status=done&style=none&taskId=ud517fa2b-8fd0-48b3-b541-aa44f194034&title=&width=1229.6)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693624797883-f7043bfd-5da2-439c-bb14-0530b9466b89.png#averageHue=%23fefdfc&clientId=u8d8a2cd9-e9e2-4&from=paste&height=454&id=u290c8e47&originHeight=568&originWidth=981&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=43470&status=done&style=none&taskId=u0728bfc8-495f-4438-b744-9c5d635018b&title=&width=784.8)
搜一下this.f502f这个变量在哪儿被赋值过
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693717714323-aadd7400-616b-4523-87ec-cdb9af093871.png#averageHue=%23f7f5f1&clientId=u7cb38088-80f0-4&from=paste&height=282&id=u5e5040f2&originHeight=353&originWidth=1550&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=57042&status=done&style=none&taskId=ubc6d78ec-2600-47d1-9497-7fe8f348380&title=&width=1240)
都给他强行赋值为true，这里有个坑：f502f是重命名后的结果，f502f的原名称是f，为了防止跟包名冲突
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693624835184-df0dd835-e83d-4d3c-bdc3-dbfd257165b1.png#averageHue=%23fefdfc&clientId=u8d8a2cd9-e9e2-4&from=paste&height=279&id=NQoAr&originHeight=349&originWidth=728&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=25737&status=done&style=none&taskId=ua2fb1d91-f292-4611-8fd4-4142a17b093&title=&width=582.4)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693718395583-51457a6e-2f4f-47ba-a9a4-e8067c43d889.png#averageHue=%23fdfdfc&clientId=u7cb38088-80f0-4&from=paste&height=555&id=u00d07f0d&originHeight=694&originWidth=1517&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=68648&status=done&style=none&taskId=u875ac0ed-b674-4cdc-8e42-57b956b5888&title=&width=1213.6)
## 最后
破解完终于可以正常启动了，可惜的是最后他这个软件还是无法使用。。。看了下评论都是在喷他不能用的，还好没有花钱买pro。。。
![-1543367649.jpeg](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696755241680-5438313a-97f5-4c87-8031-449ddbb2aa19.jpeg#averageHue=%231f5965&from=url&height=454&id=JkmYr&originHeight=1024&originWidth=1365&originalType=binary&ratio=1&rotation=0&showTitle=false&size=184769&status=done&style=none&title=&width=605)
![-1543428131.jpeg](https://cdn.nlark.com/yuque/0/2023/jpeg/1599908/1696755250075-020001f2-9ea6-4386-a298-b3bf669e72ad.jpeg#averageHue=%239398ba&from=url&height=844&id=GWQOc&originHeight=1365&originWidth=1024&originalType=binary&ratio=1&rotation=90&showTitle=false&size=129998&status=done&style=none&title=&width=633)

![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1693719236516-030b5156-6eed-4de5-8e5c-b9edc9efbf08.png#averageHue=%23fefefe&clientId=u7cb38088-80f0-4&from=paste&height=697&id=ubac5dee1&originHeight=871&originWidth=1259&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=73672&status=done&style=none&taskId=u834ccda6-0e1d-48fb-a7e0-42c9acf9253&title=&width=1007.2)
看了下代码，大概的原理是模拟了一个输入法，通过获取不同的蓝牙指令映射为不同的key。这个apk大概是5-6年前的，猜测可能是随着版本迭代key已经更新，也许更新一下最新的映射或许还能用？这部分就等后面有空了再研究吧。

