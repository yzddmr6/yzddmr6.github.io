# 精简JRE,打造无依赖的Java-ShellCode-Loader


<meta name="referrer" content="no-referrer" />



首发于公众号：**网络安全回收站**

@yzddMr6

## 前言

​	利用小众语言进行免杀一直是一个屡试不爽的方法，从python到go再到现在的nim免杀，用的人越多杀软的检测也就越来越严格。现在自己写的go程序基本只要涉及到网络通信360就干掉了。那么还有没有什么新的姿势呢？

之前介绍过在As-Exploits中用到的基于JNA实现的ShellCodeLoader(https://t.zsxq.com/022FQrFAu)，这个Loader在精简后不到1m，配合JarLoader模块在插件里面可以直接内存加载，文件不落地。后来发现落地了问题也不大，到现在VT还是0/57。所以后来抽出来作为一个单独的项目：https://github.com/yzddmr6/Java-Shellcode-Loader![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654398793581-d55b9033-7556-470c-95f1-9e8798f21679.png)

实战里面有Java的WebShell用起来非常方便，一键免杀xxx。但是缺点是如果用来钓鱼，或者碰上jdk环境过高过低都用不了，还是有局限性。所以就研究了一下怎么跟jre一起打包成一个单独的可执行文件exe。



​	目前成果如下：用自解压精简后带jre环境的exe只有6.5m，用Enigma Virtual Box压缩模式8.5m，跟python打包后差不多大小，VT 6/67，基本可以实现我们的需求。

## jre目录结构

原版一个jre大概快200m，在没有安装jre环境的普通用户来说，显然带着整个jre和后门一起打包是不可能的了，但我们可以从jre中提取加载后门时需要用到的class文件,并集合到一起,这样就能大大压缩jre的体积。

jre最主要的两个目录是bin跟lib，bin下主要是各类dll跟可执行文件，lib下是java的依赖库。精简jre就可以从这两方面入手。

### lib目录

**access-bridge-64.jar**

Java Accessibility API是Java Accessibility Utilities的一部分，它是一组实用程序类，可帮助辅助技术提供对实现Java Accessibility API的GUI工具包的访问。

**charsets.jar**

Java 字符集，包含 Java 所有支持字符的字符集

**cldrdata.jar**

Unicode CLDR为软件提供了支持世界语言的关键构建块，提供了最大和最广泛的语言环境数据库。 这些数据被广泛的公司用于其软件国际化和本地化，使软件适应不同语言的惯例以用于此类常见软件任务.

**deploy.jar**

Java安装目录的常见部分 - 该文件运行某些产品的安装。 正确设置Java路径后，用户可以执行此文件（只需双击它或按文件上的Enter键），要部署的应用程序将运行其安装程序。 例如。 诺基亚OVI套件通常使用这种部署形式。 作为彼此的JAVA包，如果您将其重命名为ZIP并打开内容，则可以检查包中的类。

**dnsns.jar**

即DNS naming service ,提供DNS地址服务的包，里面只有2个方法 getHostByAddr和 lookupAllHostAddr

**jaccess.jar**

定义Assistive Technologies.AWT（Abstract Window Toolkit）使用的JDK实用程序类

**javaws.jar**

JNLP（Java Network Launching Protocol ）是java提供的一种可以通过浏览器直接执行java应用程序的途径。

**jce.jar**

java类库是java发布之初就确定了的基础库， 而javax类库则是在上面增加的一层东西，就是为了保持版本兼容要保存原来的，但有些东西有了更好的解决方案， 所以，就加上些，典型的就是awt(Abstract Windowing ToolKit) 和swing。） 这个包都是加密相关的。

**jfr.jar**

和 jdk\bin\jmc.exe有关系。Java Mission Control 包括 JMX 控制台和 Java 飞行记录器。 Java 飞行记录器 (JFR) 是一个用于收集有关正在运行的 Java 应用程序的诊断数据和概要分析数据的工具。它集成到 Java 虚拟机 (JVM) 中， 几乎不会带来性能开销，因此甚至可以在高负载生产环境中使用。使用默认设置时，内部测试和客户反馈表明性能影响低于 1%。 对于一些应用程序，这一数字会大幅降低。但是，对于短时间运行的应用程序 (不是在生产环境中运行的应用程序类型)， 相对的启动和预热时间可能会较长，这对性能的影响可能会超过 1%。JFR 收集有关 JVM 及其上运行的 Java 应用程序的数据。

**jfxrt.jar**

JDK有个 rt.jar ，是存储JAVA语言核心类的的。这个jfxrt.jar就相当于JavaFX的rt.jar. JavaFX是一组图形和媒体包，使开发人员能够设计，创建，测试，调试和部署在不同平台上一致运行的富客户端应用程序。在jdk最新的发版当中，javafx的包已经被移除了。

**jfxswt.jar**

也是和JavaFx相关，为JavaFx和Swing提供一些兼容性操作。

**jsse.jar**

SSL连接，验证的包，

**localedata.jar**

日期显示国际化的包，里面包含各地区的日期文字。

**management-agent.jar**

里面只有一个文本文件。

**nashorn.jar**

包括

1.动态链接.包含用于链接调用的动态调用站点的接口和类。 dynalink与java.lang.invoke包密切相关，并且依赖于该包。 虽然java.lang.invoke为invoke dynamic调用站点的动态链接提供了一个低级别的API，但它不提供一种方法来表示对象的更高级别操作，也不提供实现这些操作的方法。 如果一种语言是静态类型的，并且它的类型系统与JVM的类型系统匹配，那么它可以使用通常的调用、字段访问等指令（例如invokevirtual、getfield）来实现这一点。 但是，如果语言是动态的（因此，某些表达式的类型直到在运行时进行计算时才知道），或者其对象模型或类型系统与JVM的对象模型或类型系统不匹配， 那么它应该使用invokedynamic调用站点，并让dynalink管理它们。

2.Javascript引擎 从 JDK 8 开始，Nashorn取代 Rhino 成为 Java 的嵌入式 JavaScript 引擎。Nashorn 完全支持ECMAScript 5.1 规范以及一些扩展。该特性允许开发人员将 JavaScript 代码嵌入到 Java 中，甚至从嵌入的 JavaScript 中调用 Java。此外， 它还提供了使用jrunscript从命令行运行 JavaScript 的能力。

**plugin.jar**

功能很庞大的一个包。

**resources.jar**

提示信息显示国际化的包，里面各地区的文字,图片等。

**rt.jar**

java核心源代码包

**sunec.jar ,sunjce_provider.jar,sunmscapi.jar,sunpkcs11.jar**

都是加密相关的包。

**zipfs.jar**

java 对zip文件操作的支持。

### bin目录

只找到了jdk/bin目录的介绍，jre有些没有，将就着看一下吧

| appletviewer.exe   | 用于运行并浏览applet小程序。                                 |
| ------------------ | ------------------------------------------------------------ |
| apt.exe            | 注解处理工具(Annotation Processing Tool)，主要用于注解处理。 |
| extcheck.exe       | 扩展检测工具，主要用于检测指定jar文件与当前已安装的Java SDK扩展之间是否存在版本冲突。 |
| idlj.exe           | IDL转Java编译器(IDL-to-Java Compiler)，用于为指定的IDL文件生成Java绑定。IDL意即接口定义语言(Interface Definition Language)。 |
| jabswitch.exe      | Java访问桥开关(Java Access Bridge switch)，用于启用/禁用Java访问桥。Java访问桥内置于Java 7 Update 6及以上版本，主要为Windows系统平台提供一套访问Java应用的API。 |
| jar.exe            | jar文件管理工具，主要用于打包压缩、解压jar文件。             |
| jarsigner.exe      | jar密匙签名工具。                                            |
| java.exe           | Java运行工具，用于运行.class字节码文件或.jar文件。           |
| javac.exe          | Java编译工具(Java Compiler)，用于编译Java源代码文件。        |
| javadoc.exe        | Java文档工具，主要用于根据Java源代码中的注释信息生成HTML格式的API帮助文档。 |
| javafxpackager.exe | JavaFX包装器，用于执行与封装或签名JavaFX应用有关的任务。     |
| javah.exe          | Java头文件工具，用于根据Java类生成C/C++头文件和源文件(主要用于JNI开发领域)。 |
| javap.exe          | Java反编译工具，主要用于根据Java字节码文件反汇编为Java源代码文件。 |
| java-rmi.exe       | Java远程方法调用(Java Remote Method Invocation)工具，主要用于在客户机上调用远程服务器上的对象。 |
| javaw.exe          | Java运行工具，用于运行.class字节码文件或.jar文件，但不会显示控制台输出信息，适用于运行图形化程序。 |
| javaws.exe         | Java Web Start，使您可以从Web下载和运行Java应用程序，下载、安装、运行、更新Java应用程序都非常简单方便。 |
| jcmd.exe           | Java 命令行(Java Command)，用于向正在运行的JVM发送诊断命令请求。 |
| jconsole.exe       | 图形化用户界面的监测工具，主要用于监测并显示运行于Java平台上的应用程序的性能和资源占用等信息。 |
| jdb.exe            | Java调试工具(Java Debugger)，主要用于对Java应用进行断点调试。 |
| jhat.exe           | Java堆分析工具(Java Heap Analysis Tool)，用于分析Java堆内存中的对象信息。 |
| jinfo.exe          | Java配置信息工具(Java Configuration Information)，用于打印指定Java进程、核心文件或远程调试服务器的配置信息。 |
| jmap.exe           | Java内存映射工具(Java Memory Map)，主要用于打印指定Java进程、核心文件或远程调试服务器的共享对象内存映射或堆内存细节。 |
| jmc.exe            | Java任务控制工具(Java Mission Control)，主要用于HotSpot JVM的生产时间监测、分析、诊断。 |
| jps.exe            | JVM进程状态工具(JVM Process Status Tool)，用于显示目标系统上的HotSpot JVM的Java进程信息。 |
| jrunscript.exe     | Java命令行脚本外壳工具(command line script shell)，主要用于解释执行javascript、groovy、ruby等脚本语言。 |
| jsadebugd.exe      | Java可用性代理调试守护进程(Java Serviceability Agent Debug Daemon)，主要用于附加到指定的Java进程、核心文件，或充当一个调试服务器。 |
| jstack.exe         | Java堆栈跟踪工具，主要用于打印指定Java进程、核心文件或远程调试服务器的Java线程的堆栈跟踪信息。 |
| jstat.exe          | JVM统计监测工具(JVM Statistics Monitoring Tool)，主要用于监测并显示JVM的性能统计信息。 |
| jstatd.exe         | jstatd(VM jstatd Daemon)工具是一个RMI服务器应用，用于监测HotSpot JVM的创建和终止，并提供一个接口，允许远程监测工具附加到运行于本地主机的JVM上。 |
| jvisualvm.exe      | JVM监测、故障排除、分析工具，主要以图形化界面的方式提供运行于指定虚拟机的Java应用程序的详细信息。 |
| keytool.exe        | 密钥和证书管理工具，主要用于密钥和证书的创建、修改、删除等。 |
| kinit.exe          | 主要用于获取或缓存Kerberos协议的票据授权票据。               |
| klist.exe          | 允许用户查看本地凭据缓存和密钥表中的条目(用于Kerberos协议)。 |
| ktab.exe           | Kerberos密钥表管理工具，允许用户管理存储于本地密钥表中的主要名称和服务密钥。 |
| native2ascii.exe   | 本地编码到ASCII编码的转换器(Native-to-ASCII Converter)，用于"任意受支持的字符编码"和与之对应的"ASCII编码和(或)Unicode转义"之间的相互转换。 |
| orbd.exe           | 对象请求代理守护进程(Object Request Broker Daemon)，它使客户端能够透明地定位和调用位于CORBA环境的服务器上的持久对象。 |
| pack200.exe        | JAR文件打包压缩工具，它可以利用Java类特有的结构，对普通JAR文件进行高效压缩，以便于能够更快地进行网络传输。 |
| packager.exe       | 这是微软提供的对象包装程序，用于对象安装包。                 |
| policytool.exe     | 策略工具，用于管理用户策略文件(.java.policy)。               |
| rmic.exe           | Java RMI 编译器，为使用JRMP或IIOP协议的远程对象生成stub、skeleton、和tie类，也用于生成OMG IDL。 |
| rmid.exe           | Java RMI 激活系统守护进程，rmid启动激活系统守护进程，允许在虚拟机中注册或激活对象。 |
| rmiregistry.exe    | Java 远程对象注册表，用于在当前主机的指定端口上创建并启动一个远程对象注册表。 |
| schemagen.exe      | XML schema生成器，用于生成XML schema文件。                   |
| serialver.exe      | 序列版本命令，用于生成并返回serialVersionUID。               |
| servertool.exe     | Java IDL 服务器工具，用于注册、取消注册、启动和终止持久化的服务器。 |
| tnameserv.exe      | Java IDL瞬时命名服务。                                       |
| unpack200.exe      | JAR文件解压工具，将一个由pack200打包的文件解压提取为JAR文件。 |
| wsgen.exe          | XML Web Service 2.0的Java API，生成用于JAX-WS Web Service的JAX-WS便携式产物。 |
| wsimport.exe       | XML Web Service 2.0的Java API，主要用于根据服务端发布的wsdl文件生成客户端存根及框架 |
| xjc.exe            | 主要用于根据XML schema文件生成对应的Java类。                 |

## 精简rt.jar

rt.jar是java核心源代码包，原版有61m，我们主要的精简也就是从这里入手。原理是jar包运行是加上-XX:+TraceClassLoading参数可以打印出所有被加载过的class文件，然后在对这部分class进行二次打包，生成我们的精简rt.jar。这里借MG师傅的图一用：

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654416377870-a363d4f7-9623-4a27-b510-f3aeb6869f53.png)

代码是参考MG1937师傅的这篇文章，https://www.cnblogs.com/aldys4/p/14879607.html，修改后代码如下：

```plain
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.List;

public class Main {
    public static void main(String[] arg) throws IOException {
        Runtime runtime = Runtime.getRuntime();
        String[] command = {"java", "-jar", "-XX:+TraceClassLoading", "D:\\ShellcodeLoader_jar\\ShellcodeLoader.jar", "aaaa"}; //这里要加上参数
        Process process = runtime.exec(command);
        BufferedReader bReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        StringBuffer sBuffer = new StringBuffer();
        List<String> list = new ArrayList<String>();
        int i = 0;
        String lineString;
        while ((lineString = bReader.readLine()) != null) {
            String core = getCore(lineString);
            if (core != "") {
                sBuffer.append("\n" + core);
                list.add(getCore(lineString.replace(".", "/")));
            }

            i++;
        }
        bReader.close();
        System.out.println(sBuffer.toString());
        list.add(0, "D:\\rt.jar");
        list.add(0, "xvf");
        list.add(0, "jar");
        String[] jar = list.toArray(new String[list.size()]);
        process = runtime.exec(jar);
        getOutput(process);
        System.out.println("Load class:" + i);
        System.out.println("jar xvf done!");
        String[] cmdJarPackage = cmd("jar cvf rt.jar com java javax META-INF org sun sunw");
        runtime.exec(cmdJarPackage);
        System.out.println("All done!");
    }

    public static String getCore(String line) {
        String result = null;
        if (line.startsWith("[Loaded")) {
            if (line.indexOf(".jna.") > 0 || line.indexOf("asexploits") > 0) {
                return "";//过滤jna包跟我们自己的包名

            } else {
                result = line.split(" ")[1];
            }
            return result;
        } else {
            return "";
        }
    }

    public static String[] cmd(String cmd) {
        return cmd.split(" ");

    }

    public static void getOutput(Process process) throws IOException {
        BufferedReader bReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        while (bReader.readLine() != null) {
            System.out.println("\n" + bReader.readLine());
        }
    }
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654323856680-28f4c1e4-b096-4f5b-ae99-7871ea27aede.png)

这里有一个坑点，就是第一次获取加载的class信息的时候没有加上ShellCode参数，也就会导致有些运行期间才会用到的类没有加载。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654323832072-4bec73b4-052b-4bef-8523-007c7511b151.png)

解决办法就是加上要执行的ShellCode，把执行过程中所有加载的类都暴露出来，然后再打包。这里的aaaa随便写，只要能走到注入的过程就可以。可以看到现在的rt.jar已经不报错了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654323841335-fa2e460a-8a61-48e2-934c-35c6d1446ceb.png)

精简之前lib目录是104m，现在已经压缩到44m了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654421913746-d773e9bf-a180-4228-9fa8-2a049bb11096.png)

我们压缩后的rt.jar也只有1.8m大小

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654416571886-1fbe5d4e-1638-4464-bc70-29a437d2d11b.png)

这样肯定还是不够的，剩下目录里也有很多冗余的文件，这部分基本直接删除就可以了，不需要二次打包。另外charsets.jar还是有优化空间的，可以用类似rt.jar的方法进行精简，这里就懒得处理了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654416689586-d3603a9d-a495-43c4-9fb5-91fc7603f2e2.png)

最后的lib只有5m大小了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654421892797-8f122be0-c365-4f35-94dc-65c92fd89f48.png)

## 精简dll

bin目录同样很大，也是我们要优化的对象。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654416802258-90e5025f-8cc2-4b54-864c-53c0b6bb8c9b.png)

这里采取的办法是用process explorer查看程序运行时加载了哪些dll或者exe，仅保留这部分，其他的都可以删掉。

注意这里最好加一个System.in.read()保持控制台不退出，不然就会一闪而过，process explorer就看不到了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654324241719-9631757f-45cd-490a-b440-46ef1d99c987.png)

还有一个简单的办法，在程序跑起来的同时，删除bin目录下所有文件，如果提示被占用了那么就是被打开了，把这部分跳过即可。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654324404130-9d6322ae-56c7-4d8e-99c9-234a7b26b177.png)

精简过后的bin目录大概10m，主要是jvm.dll比较大。他是jvm的核心链接库，不能轻易改动。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654417001774-91ed03f7-cf23-4804-8882-e1d8e5ebd0d2.png)

## 自解压捆绑执行

环境整好了，接下来就是让他跑起来。自解压是钓鱼老套路了，搞个vbs来运行我们的jar。这里的ShellcodeLoader.jar我硬编码了一个弹计算器的ShellCode先测试一下。

```plain
Set ws = CreateObject("Wscript.Shell")
ws.run "cmd /c .\jre\bin\java.exe -jar .\ShellcodeLoader.jar"
```

执行成功

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654398350157-8a74dd70-2a5d-4764-b2f6-be51c99d81b3.png)

压缩出来6.5m

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654398544508-4b172722-7ee1-4267-8af4-fb2a54cfffaf.png)

VT 6/62还好，但是360杀了，估计自解压这种已经进特征库了。

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654400461938-b4e0c723-98aa-478e-9c81-9a69d03dfb91.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654399470289-91f1735f-f120-439e-868d-6a2317961d45.png)

## EnigmaVirtualBox打包全部文件

自解压估计已经被重点监控了，用EnigmaVirtualBox把jre跟jar打包成一个单独的exe试试

这里install.exe是偷懒用msfvenom -p windows/exec生成的

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654396931823-6e4e0058-cb09-4394-b4f1-401be402ce0a.png)



可以执行，但是会有UAC提示框，不太行

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654397277906-b7522dc2-cb74-4a85-beea-fe384d1bd75d.png)

VT上查杀过半了，看来不能偷懒

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654399622394-7ce6bd30-dd17-4c15-ae18-185f951d8695.png)

后来又用C++写了一个exe去调用，还是杀的比较多

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654418514301-dcd3f489-e4de-44f3-b6af-2879f53d9c2a.png)



## EnigmaVirtualBox打包jre

不过话说我为什么要打包到一起呢，沙箱里面一跑就出来了，这就失去了jar的优势：jar除了可以分离真正的Payload以外，本身就可以加各种混淆，各种商业软件也都是带混淆的，杀软也不能直接杀。

转换思路，我可以仅打包一个人畜无害的jre到exe，然后再jre.exe -jar xxx去调用。

打包方法同上，打包出来后8.2m，测试一下能不能用

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654400605080-580279a2-54a9-48ef-8d4a-b3bb03d8339c.png)

执行成功

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654400755143-df7a8150-d429-4a01-88de-5d895969e7a9.png)

xxx不杀

![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654419035627-ccd04d7b-ea01-4910-8541-a797cd5a4256.png)

VT测一下还有6个引擎检出。。。这tm就是个java.exe啊，还Static ML，真就瞎告呗。这样说我也能搞一个杀毒引擎，看到PE头就杀，名字就叫Deep Static ML。



![img](https://cdn.nlark.com/yuque/0/2022/png/1599908/1654418935656-47799414-40b3-4c03-a688-92305cb83889.png)

## 最后

本文仅用于安全研究，请勿用于非法用途。如果有什么问题欢迎交流。
