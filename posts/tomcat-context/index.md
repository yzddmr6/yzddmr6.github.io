# 关于Tomcat中的三个Context的理解


<meta name="referrer" content="no-referrer" />

p牛在知识星球里问了一个问题：Tomcat中这三个StandardContext、ApplicationContext、ServletContext都是干什么的

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615783546397-3d81b048-fdc8-47a9-b395-1dac64638e0a.png)

skay师傅给出了自己的理解：https://mp.weixin.qq.com/s/BrbkTiCuX4lNEir3y24lew

这里来讲一讲我的理解，说的不一定对，仅供参考。

### Context

context是上下文的意思，在java中经常能看到这个东西。那么到底是什么意思呢？

根据我的理解，如果把某次请求比作电影中的事件，那么context就相当于事件发生的背景。例如一部电影中的某个镜头中，张三大喊“奥利给”，但是只看这一个镜头我们不知道到底发生了什么，张三是谁，为什么要喊“奥利给”。所以就需要交代当时事情发生的背景。张三是吃饭前喊的奥利给？还是吃饭后喊的奥利给？因为对于同一件事情：张三喊奥利给这件事，发生的背景不同意义可能是不同的。吃饭前喊奥利给可能是饿了的意思，吃饭后喊奥利给可能是说吃饱了的意思。在WEB请求中也如此，在一次request请求发生时，背景，也就是context会记录当时的情形：当前WEB容器中有几个filter，有什么servlet，有什么listener，请求的参数，请求的路径，有没有什么全局的参数等等。

### ServletContext

ServletContext是Servlet规范中规定的ServletContext接口，一般servlet都要实现这个接口。大概就是规定了如果要实现一个WEB容器，他的Context里面要有这些东西：获取路径，获取参数，获取当前的filter，获取当前的servlet等

```
package javax.servlet;

...

public interface ServletContext {
    String TEMPDIR = "javax.servlet.context.tempdir";
    String ORDERED_LIBS = "javax.servlet.context.orderedLibs";

    String getContextPath();

    ServletContext getContext(String var1);

...

    /** @deprecated */
    @Deprecated
    Servlet getServlet(String var1) throws ServletException;

    /** @deprecated */
    @Deprecated
    Enumeration<Servlet> getServlets();

    /** @deprecated */
    @Deprecated
    Enumeration<String> getServletNames();

    void log(String var1);

    /** @deprecated */
    @Deprecated
    void log(Exception var1, String var2);

    void log(String var1, Throwable var2);

    String getRealPath(String var1);

    String getServerInfo();

    String getInitParameter(String var1);

    Enumeration<String> getInitParameterNames();

    boolean setInitParameter(String var1, String var2);

    Object getAttribute(String var1);

    Enumeration<String> getAttributeNames();

    void setAttribute(String var1, Object var2);

    void removeAttribute(String var1);

    String getServletContextName();

    Dynamic addServlet(String var1, String var2);

    Dynamic addServlet(String var1, Servlet var2);

    Dynamic addServlet(String var1, Class<? extends Servlet> var2);

    Dynamic addJspFile(String var1, String var2);

    <T extends Servlet> T createServlet(Class<T> var1) throws ServletException;

    ServletRegistration getServletRegistration(String var1);

    Map<String, ? extends ServletRegistration> getServletRegistrations();

    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, String var2);

    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, Filter var2);

    javax.servlet.FilterRegistration.Dynamic addFilter(String var1, Class<? extends Filter> var2);

    <T extends Filter> T createFilter(Class<T> var1) throws ServletException;

    FilterRegistration getFilterRegistration(String var1);

    Map<String, ? extends FilterRegistration> getFilterRegistrations();

    SessionCookieConfig getSessionCookieConfig();

    void setSessionTrackingModes(Set<SessionTrackingMode> var1);

    Set<SessionTrackingMode> getDefaultSessionTrackingModes();

    Set<SessionTrackingMode> getEffectiveSessionTrackingModes();

    void addListener(String var1);

    <T extends EventListener> void addListener(T var1);

    void addListener(Class<? extends EventListener> var1);

    <T extends EventListener> T createListener(Class<T> var1) throws ServletException;

    JspConfigDescriptor getJspConfigDescriptor();

    ClassLoader getClassLoader();

    void declareRoles(String... var1);

    String getVirtualServerName();

    int getSessionTimeout();

    void setSessionTimeout(int var1);

    String getRequestCharacterEncoding();

    void setRequestCharacterEncoding(String var1);

    String getResponseCharacterEncoding();

    void setResponseCharacterEncoding(String var1);
}
```

### ApplicationContext

在Tomcat中，ServletContext规范的实现是ApplicationContext，因为门面模式的原因，实际套了一层ApplicationContextFacade。关于什么是门面模式具体可以看[这篇文章](https://www.runoob.com/w3cnote/facade-pattern-3.html)，简单来讲就是加一层包装。

其中ApplicationContext实现了ServletContext规范定义的一些方法，例如addServlet,addFilter等

### StandardContext

StandardContext存在于org.apache.catalina.core.StandardContext。

实际上研究ApplicationContext的代码会发现，ApplicationContext所实现的方法其实都是调用的this.context中的方法

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791333561-80d3e967-f36a-4c49-a611-a329bdf1349b.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791389467-3fe1e723-84d1-4e8b-8dfb-8f5712665a6d.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791403712-f22001f0-8c10-4bb4-9ab9-7bc1fdbe8650.png)

而这个this.context就是一个实例化的StandardContext对象。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615791137362-cd302e98-fe22-468f-ae9e-4f2085848df3.png)

所以在我看来，StandardContext是Tomcat中真正起作用的Context，负责跟Tomcat的底层交互，ApplicationContext其实更像对StandardContext的一种封装。

用下面这张图来展示一下其中的关系


![image](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615790929311-f1c15d6e-c317-41c2-9ea7-eadc91a691cf.png)





回过头看内存马。以添加filter为例，从上面的分析我们可以知道ApplicationContext跟Standerdcontext这两个东西都有addFilter的方法。那么实际选用哪一个呢？其实两种办法都可以。三梦师傅在[基于tomcat的内存 Webshell 无文件攻击技术](https://xz.aliyun.com/t/7388)这篇文章里是利用反射修改了Tomcat的LifecycleState，绕过限制条件调用的ApplicationContext中的addFilter方法。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615795697048-8b5ba421-eb1d-45a9-8084-04127e0484a5.png)

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1599908/1615796887459-f6e8da3c-9941-418c-a02e-5d217b199aa6.png)

但是因为实际上最终调用的还是StandardContext的addFilter方法，所以我们就可以直接调用StandardContext的addFilter方法进行绕过，从而省去了绕过一堆判断的过程。这种实现具体可以看这个师傅的[公众号文章](https://mp.weixin.qq.com/s/nPAje2-cqdeSzNj4kD2Zgw)。
