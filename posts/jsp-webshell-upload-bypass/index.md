# 星球问答：一次jsp上传绕过的思考


<meta name="referrer" content="no-referrer" />

## 背景

前几天有个小伙伴做项目的时候遇到一个问题来问我，大概情况如下：

1. jsp的站，可以任意文件上传
2. 上传jsp会把<%中的<给转义掉
3. 上传jspx会把<jsp:scriptlet>到</jsp:scriptlet>中的内容替换为空

问有什么突破办法？

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610679190674-baace7f2-e763-4cb2-8695-bed0661fc1e5.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610679220546-2dfa9031-8abc-4f16-a4db-2a6b3e0c2c1d.png)



当时研究了一下后jsp和jspx各给了一个解决方案，后来发到星球里后@hosch3n师傅又提出了一种新的方案，tql

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1616582626934-e9fd6f02-24b0-47b6-b814-22c972c359b9.png)

## 解决方案

其实问题等价于：

1. jsp不使用<% %>标签如何执行命令
2. jspx不使用<jsp:scriptlet> </jsp:scriptlet>如何执行命令



### jsp利用EL表达式绕过

jsp是默认解析el表达式的，并且在没有jsp标签的情况下也可以直接执行，这样就可以绕过jsp的限制。

星球里面@Gh0stFx也提到了这一点

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610679796286-4312913e-9ca9-42ba-9ba0-20352e9572a6.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610679816568-bfe65530-8ece-40ee-b6af-e4228373fa92.png)

### jspx利用命名空间绕过

因为jspx实际上是jsp的xml写法，所以继承了xml的所有特性，例如cdata跟html实体编码等，同样也继承了命名空间的特性。

https://www.runoob.com/xml/xml-namespaces.html

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610680285750-0f35e6cf-e02b-4a65-8c75-f6170768fbfd.png)

在<jsp:scriptlet>这个标签中，jsp就是默认的命名空间，但是实际上可以随意替换成其他名字

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610680407208-7224f65a-665e-413f-8f4e-b6a90fea8c4f.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610680395767-7be98260-0a6d-4bd1-b5f7-8ef901c0b6dc.png)

这样就绕过了对<jsp:scriptlet>的过滤

### jspx利用<jsp:expression>绕过

在jsp中可以利用表达式绕过，那么jspx中同样也可以，以下是jsp跟jspx语法的对照：



|                       | JSP语法        | JSP document语法                       |
| --------------------- | -------------- | -------------------------------------- |
| Page Directive        | <%@ page %>    | <jsp:directive.page />                 |
| Include Directive     | <%@ include %> | <jsp:directive.include />              |
| Tag Library Directive | <%@ taglib %>  | xmlns:prefix=”Library URI”             |
| Declartion            | <%! … %>       | <jsp:declaration> … </jsp:declaration> |
| Scriplet              | <% … %>        | <jsp:scriptlet> … </jsp:scriptlet>     |
| Expression            | <%= … %>       | <jsp:expression> … </jsp:expression>   |
| Comment               | <%-- … --%>    | <!-- … -->                             |

这个方法是@hosch3n师傅提出来的，把表达式写到jspx中，同样可以达到执行命令的目的

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610680526168-55b3cb88-20e0-42b6-8e8a-d636a19d3df0.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1610680545282-40414186-2ba2-4c30-aca0-ba88fb66946d.png)
