# 从ChatGPT到SharpAlternativeShellcodeExec


<meta name="referrer" content="no-referrer" />



## 背景

利用ShellCode进行免杀是一种最常见的免杀方式，但是常见的VirtualAlloc、CreateRemoteThread这些Windows API已经被各大杀软重点监控。那么与之相对的绕过的办法就是利用一些小众的Windows API，这些API函数往往提供了回调的功能，当它的参数是指针类型的话就可以直接执行内存当中的ShellCode，这样就绕过了敏感函数识别达到执行ShellCode的目的。



最近研究了一下https://github.com/aahmad097/AlternativeShellcodeExec这个项目，该项目提供了很多绕过的API函数。在渗透测试的过程中为了逃避检测，我们常常希望实现内存加载，无文件攻击。原项目是C++写的，PE转ShellCode等方法有时候并不稳定。而C#可以直接内存加载，并且从Windows XP以来每台Windows上都默认安装了.NET，所以就萌生了用C#重写一遍的方法，同时也可以加深对项目的理解。



但是C#调用Windows API的时候需要额外进行函数的声明，并且要实现C++类型到C#类型的转化，这部分的工作十分的繁琐，也不感兴趣。所以就想到了用ChatGPT帮我去做转化。

## 初步尝试

先随便找了一个回调改写试试，发现就算是硬编码ShellCode+裸的API调用，C#重写后的版本也比原版本少20个左右引擎的检出。可能是原来的项目已经被各大厂商都加过规则了，C#版本的还没有，看起来有搞头。

C++版本

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682324049390-04a0c9ed-34c1-4431-b90d-01a8666cd468.png)

C#版本

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682324077632-bd2ac178-46ef-4f6e-bf41-c615577f75a1.png)



## 调教ChatGPT

不得不说ChatGPT确实很牛逼，本来只是想让他转换一个函数声明，但是最后几乎替我完成了一大半的重写工作（失业警告）。少部分情况下一次就能转换出能运行的代码，大多数情况我们也只需要对生成的代码进行微调即可。不过GPT偶尔也有自作聪明，胡说八道的情况，这个时候就需要调整我们的prompt来一步步引导他。

### 替换等价函数

最开始的咒语很简单：

将以下代码用C#重写:xxxx，但是ChatGPT有时候会自作聪明的把我们想要触发的回调函数进行等价替换了，这样也就达不到我们的效果。

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681888020204-7496d762-ab2d-4c94-ba72-4158bff8a656.png)



![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681888038423-334ef0db-c7ea-46b6-932b-1cf2194924ed.png)

### 错误的DllImport提示

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681897240446-6c7f1f3c-5052-4f1a-89e4-ba4563bf4bc9.png)

GPT认为EnumICMProfilesW是在mscms.dll中，但是实际上这个函数是存在于gdi32.dll中的：

https://learn.microsoft.com/en-us/windows/win32/api/wingdi/nf-wingdi-enumicmprofilesw

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682641128146-f0cb0abd-783c-4902-9dc2-875197e60602.png)

### 错误的参数位置

在重写EnumDesktopWindows这个回调函数的时候，ChatGPT一直给出了错误的参数调用，用于回调的addr指针应该放到倒数第二个参数，但是ChatGPT一直认为要放到倒数第一个参数，怎么提示也没用，后来自己改了。

```plain
using System;
using System.Runtime.InteropServices;

class Program
{
    [DllImport("kernel32.dll", SetLastError = true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("ntdll.dll")]
    static extern void RtlMoveMemory(IntPtr dest, byte[] src, uint length);

    [DllImport("user32.dll")]
    static extern bool EnumDesktopWindows(IntPtr hDesktop, EnumWindowsProc lpfn, IntPtr lParam);

    delegate bool EnumWindowsProc(IntPtr hWnd, IntPtr lParam);

    static bool MyCallback(IntPtr hWnd, IntPtr lParam)
    {
        // Do something with hWnd
        return true;
    }

    static void Main(string[] args)
    {
        // alfarom256 calc shellcode
        byte[] op = new byte[] { };

        IntPtr addr = VirtualAlloc(IntPtr.Zero, (uint)op.Length, 0x1000, 0x40);
        RtlMoveMemory(addr, op, (uint)op.Length);

        if (addr != IntPtr.Zero)
            EnumDesktopWindows(GetThreadDesktop(GetCurrentThreadId()), new EnumWindowsProc(MyCallback), addr);
    }

    [DllImport("user32.dll")]
    static extern IntPtr GetThreadDesktop(uint dwThreadId);

    [DllImport("kernel32.dll")]
    static extern uint GetCurrentThreadId();
}
```

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681954962876-75d36481-6a27-40d3-be4e-0b67fcfbdf8c.png)

重写后的代码也是错误的。



## 整体感受

讲完了ChatGPT的坑，其实总体体验下来感觉还是不错的，遇到不熟悉的API可以直接问他，体验比去找文档好的多。

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682329527778-65d65f02-13c6-4675-8081-7cbe9bf3e4a1.png)



并且不仅会给出转换后的代码，并且还会告诉你代码的大体逻辑，以及需要注意的点。

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681954857839-a5eec885-2818-4623-9356-a9403c6ec952.png)

代码逻辑的梳理

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681982079207-c12171f4-2a29-4e0b-a951-1b603173a8e3.png)



遇到报错贴给他，他也会给出一定的解决方案，当然还是需要人工检验的。

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1681982335141-a8a06b9a-a4b0-4083-ae68-d721d929ca33.png)

利用零零碎碎的时间，终于把45个Project都重写完了，在此期间也真正体会到了ChatGPT在提升生产力中的应用。



## 进一步优化

在有思路的情况下，还可以用ChatGPT进一步提高免杀效果。随便改一下做个例子：

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682324948123-283481ce-67fc-41c2-b7e9-058bc2650369.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682324967984-56ca1870-2823-4dc9-b8b6-27f7d2104de1.png)

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682065415352-bb4716f6-fe97-40ce-af94-e14797afb481.png)



![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682061253916-feea7c82-3832-41f0-bc0c-f113223f489a.png)



![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682064903931-bee70de3-c95d-42ba-8874-6244ad4c9a61.png)

最后的代码也是可用的，弹个计算器

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682324827661-c131e66d-9be4-4774-a0d4-e294183aabd5.png)



## 无法运行的FiberContextEdit

在重写后的45个项目中，利用FiberContextEdit方法的始终无法正常运行。问了ChatGPT后应该是跟C++中函数转换到C#后偏移量不同有关，限制于水平问题没有深入研究，有知道原因的同学可以私下里交流交流

![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682412684662-42d15d77-d418-43e8-ae9b-813f7b78c1a5.png)



![img](https://cdn.nlark.com/yuque/0/2023/png/1599908/1682412074180-2b7efc3b-6625-47f3-9104-60d4605de6f1.png)

项目代码已经同步到我的github，如果有问题欢迎反馈：

https://github.com/yzddmr6/SharpAlternativeShellcodeExec

