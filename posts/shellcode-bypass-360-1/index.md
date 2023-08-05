# MSF免杀360+火绒上线(一)


<meta name="referrer" content="no-referrer" />

## 前言

venom前一段时间更新了，增加了一些新的免杀上线方式。

人怕出名猪怕壮，原来的混淆免杀对于360来说已经全部失效了

所以本菜鸡就研究了一下免杀。

自己二进制方面就是弟弟一个，如果有什么说的不对的地方还请联系我指出，感激不尽。

## 正文

### 测试信息

系统环境：kali，Win7 x64

测试杀软：   360+火绒 防护全开

免杀工具： venom

虽然360是个大流氓，但是不得不说很厉害，毕竟有时候只能用流氓来打败流氓，实际中也遇到的比较多。

### backdoor生成

把venom原来的payload基本试了一遍，发现要么进了360的特征库，要么就是一执行就被杀。

但是msbuild还是稳得一批

关于msbuild上线的用法可以看三好学生大佬的这一篇文章 [Use MSBuild To Do More](https://3gstudent.github.io/3gstudent.github.io/Use-MSBuild-To-Do-More/)

更新好venom后，选择windows，然后选择19

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416357-9113dbfb-6c50-4d2f-97c7-9b073737cef9.png)

尽量使用加密流量传输，例如https或者rc4

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416521-ed76f67a-f124-4ff4-9ae2-c2e069aae0c4.png)

选择好后venom会自动生成shellcode

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416607-8039ca33-97cf-49ad-bf4a-affc0487d022.png)

就可以看到加有我们shellcode的xml文件了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416702-dab7ab89-7c85-4586-8abc-5d3b6e353040.png)

然后通过python的http服务来下载

可以看到没有报毒，提示安全

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416821-8a30167c-4171-4de7-b76a-744a0ffd64b9.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416928-f80c397c-815b-47b3-a7a0-ac4ef630341f.png)

### 自动安装net4.0

win7默认是不带net4.0的，所以生成的backdoor是无法直接运行的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417028-fead25c2-1cac-4227-8db0-70d66da3400e.png)

三好学生大佬的博客上给出了[命令行下静默安装net4.0的方法](https://3gstudent.github.io/3gstudent.github.io/渗透基础-命令行下安装Microsoft_.NET_Framework/)

开始编译的时候报了一大堆错

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417162-7ece5434-a944-42f2-ad9b-5d1c63ee9378.png)

这里有几个坑需要解决

- 版本必须用 release
- 字符集必须用 unicode

- 必须设置 在共享DLL中使用MFC

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417256-e5987e27-0820-4cdc-8dda-70828db0dd41.png)

奇怪的是我明明已经下载好了net 4.0的安装包并且放到同一目录下面，但是静默下载工具一直提示找不到setup file的路径，后来换绝对路径也不行，如果有知道解决办法的表哥麻烦告诉我一下。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417339-4da85d34-d3db-498a-ae46-c01b61629efb.png)

最后本弟弟还是决定手动给他装上。。。。

### 绕过360火绒上线

执行bat后就会弹回来一个shell

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417441-5db4d05e-3c2f-4adc-83bd-ae2e8277b764.png)

ps看到有360跟火绒的防御进程，但是完全是静默上线

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417532-45d32899-8a10-465e-a87a-845735343477.png)

也可以监控屏幕，执行命令等

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417633-49cb4811-31dc-4b79-9ee7-e6de69a6bcc8.png)

### 要解决的问题

其实可以看到我们只是administrator权限，使用getsystem提示失败

装的老版本的win7，并没有打补丁，但是各种bypass uac跟溢出提权都GG

原因是，虽然上线的shellcode是免杀的，但是你提权之后会执行一个默认的payload去反弹shell，这时候反弹shell的payload并不是免杀的，就会被杀软拦截。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900417836-ec13b06e-67bb-4871-806d-e8b890bf33c0.png)

因为默认是反弹一个reverse_tcp的shell，我想着如果用windows/exec去cmd再去执行一次我们免杀的xml是不是就能返回一个system权限了呢

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418008-036e7794-4ac0-452d-a9af-b4497697d875.png)

结果还是GG。。。试了很多办法也没解决。。。

等我研究一下再来给大家分享吧。。。

留下了没有技术的泪水.jpg

## 最后

TideSec最近出的这一个系列不错，大家有空了可以看一看

https://github.com/TideSec/BypassAntiVirus
