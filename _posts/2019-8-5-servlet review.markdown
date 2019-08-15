---
layout: post
title:  servlet review
date:   2019-8-15 14:32:00
categories: java web servlet
---
### 1.写在前面
java 做web这一块，现在已经非常成熟了，框架封装也是非常完善，不少新手也是直接从springmvc，struts2，jsf等框架开始。甚至我自己工作中一直接触的，都是struts2，springmvc，springboot这些封装到牙齿的东西。java web最本质的东西，时间久了也生疏了很多。在这里理一下。

### 2. Servlet
#### 1. servlet先关的几个包和接口(类)
java web其实就一个最核心的东西，Servlet。
与他相关的有这么几个包，这么几个接口和类
```java
javax.servlet
javax.servlet.http
javax.servlet.annotation
Javax.servlet.descriptor
```
interface有:
1. Serlvet 
2. ServletRequest
3. ServletResponse，
4. ServletContext
5. ServletConfig
6. RequestDispatcher
7. Filter 

类有:
1. GenericServlet
>public abstract class GenericServlet implements Servlet, ServletConfig,java.io.Serializable

#### 2.servlet于servlet容器

**Servlet接口是Servlet和Servlet容器(jetty,tomcat)之间的一个契约**，这个契约归结起来是说: 
1. Servlet容器会把Servlet类加载到内存中，并在Servlet实例中调用特定的方法（service方法）。
2. 在一个应用中每个Servlet类型只有一个实例。
3. 用户请求会引发Servelt调用的service方法，并给这个方法穿日一个ServletRequest实例和ServletResponse实例。ServletRequest封装Http请求，ServletResponse表示当前Http请求响应。
4. Servlet容器还为**应用程序**创建一个ServeltContext实例。这个对象封装应用程序的环境细节(所以servletContext是application界别的)。
5. 容器为每个**Servlet实例**封装一个ServletConfig，ServletConfig存放配置信息，用来初始化Servlet。(所以ServeltConfig是Servelt级别的)


#### 3. Servlet接口
```java
package javax.servlet;

import java.io.IOException;

public interface Servlet {
   public void init(ServletConfig config) throws ServletException;
   public ServletConfig getServletConfig();
   public void service(ServletRequest req, ServletResponse res)
           throws ServletException, IOException;
   public String getServletInfo();
   public void destroy();
} 
```
**一个应用程序中所有线程都使用一个Servlet实例，要注意 "线程安全"**


#### 4.一个Servlet例子

##### 1. Servelt实现
可以使用注解 @WebServlet 来配置servlet
```java
@WebServlet(name="ServletConfigDemoServlet",
   urlPatterns = {"/servletConfigDemo"},
       initParams = {
       @WebInitParam(name="admin",value="junjun"),
       @WebInitParam(name="email",value="530@qq.com")
}
)
public class ServletConfigDemoServlet implements Servlet {

   transient private ServletConfig config;
   public void init(ServletConfig config) throws ServletException {
       this.config = config;
   }

   public ServletConfig getServletConfig() {
       return this.config;
   }

   public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
       String servletName = this.getServletInfo();

       ServletConfig servletConfig = getServletConfig();
       // @WebInitParam中配置的参数
       String admin = servletConfig.getInitParameter("admin");
       String email = servletConfig.getInitParameter("email");

       res.setContentType("text/html");

       PrintWriter writer = res.getWriter();
       StringBuffer sb = new StringBuffer();
       sb.append("<html><head></head>");
       sb.append("<body>hello from "+servletName);
       sb.append("<br/> admin : "+ admin );
       sb.append("<br/> email : "+ email );
       sb.append("</body></html>");
       writer.print(sb.toString());
   }

   public String getServletInfo() {
       return this.getClass().getSimpleName();
   }

   public void destroy() {
       System.out.println(this.getServletInfo()+"  destroy");
   }
}

```

##### 2.部署servlet

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/servlet-deploy1.jpg)


Servlet部署在servlet容器中，安装一个Servlet容器(比如tomcat)，下载tomcat。
Servlet的目录结构是有要求的，如下：  
1. 放在WEB-INF以外的可以由用户url访问，比如图片，jsp等。
2. 放在WEB-INF目录中的内容只能由servlet访问，不能由用户url直接访问。
3. classes中放与servlet类和其他相关类都放这里。
4. lib目录中放需要的库文件。
5. web.xml存放整个应用(servlet)的配置信息，也可以使用 @WebServlet注解来配置
如果有文件想要servlet访问，而不想要用户直接访问，就可以放在WEB-INF目录中



### 3. HttpServlet
最常用的协议还是Http，HttpServlet是Servlet实现，用来处理http请求比Servelt方便得多
HttpServlet相关的类增加几个类:
1. Cookie 和 HttpSession
2. HttpServletRequest
3. HttpServletResponse

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/servlet-httpservlet.png)


#### 1. HttpServlet的实现
源代码如下：
```java
public abstract class HttpServlet extends GenericServlet{
public void service(ServletRequest req, ServletResponse res)
   throws ServletException, IOException
{
   HttpServletRequest  request;
   HttpServletResponse response;
  
   if (!(req instanceof HttpServletRequest &&
           res instanceof HttpServletResponse)) {
       throw new ServletException("non-HTTP request or response");
   }

   request = (HttpServletRequest) req;
   response = (HttpServletResponse) res;

   service(request, response);
}


protected void service(HttpServletRequest req, HttpServletResponse resp)
   throws ServletException, IOException
{
   String method = req.getMethod();

   if (method.equals(METHOD_GET)) {
       long lastModified = getLastModified(req);
       if (lastModified == -1) {
           // servlet doesn't support if-modified-since, no reason
           // to go through further expensive logic
           doGet(req, resp);
       } else {
           long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
           if (ifModifiedSince < lastModified) {
               // If the servlet mod time is later, call doGet()
               // Round down to the nearest second for a proper compare
               // A ifModifiedSince of -1 will always be less
               maybeSetLastModified(resp, lastModified);
               doGet(req, resp);
           } else {
               resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
           }
       }

   } else if (method.equals(METHOD_HEAD)) {
       long lastModified = getLastModified(req);
       maybeSetLastModified(resp, lastModified);
       doHead(req, resp);

   } else if (method.equals(METHOD_POST)) {
       doPost(req, resp);
      
   } else if (method.equals(METHOD_PUT)) {
       doPut(req, resp);
      
   } else if (method.equals(METHOD_DELETE)) {
       doDelete(req, resp);
      
   } else if (method.equals(METHOD_OPTIONS)) {
       doOptions(req,resp);
      
   } else if (method.equals(METHOD_TRACE)) {
       doTrace(req,resp);
      
   } else {
       //
       // Note that this means NO servlet supports whatever
       // method was requested, anywhere on this server.
       //

       String errMsg = lStrings.getString("http.method_not_implemented");
       Object[] errArgs = new Object[1];
       errArgs[0] = method;
       errMsg = MessageFormat.format(errMsg, errArgs);
      
       resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
   }
}


}

```

HttpServlet 继承自GenericServlet， 重载了
```java
public void service(ServletRequest req, ServletResponse res)，
```
然后调service方法时候将ServletRequest转化为HttpServletRequest
在调用**String method = req.getMethod();** 获取http请求类型(get post 等)，调用对应的方法(get,post等)。所以我们继承HttpServlet之后只需要实现doGet，doPost等方法即可。


#### 2. HttpServlet写服务
1. 代码WelcomeServlet
只用实现doGet方法，我们没有使用@WebServlet来配置servlet，就需要在web.xml中来配置
```java
public class WelcomeServlet extends HttpServlet {

   @Override
   public void doGet(HttpServletRequest req, HttpServletResponse res)
           throws ServletException, IOException {
       PrintWriter writer = res.getWriter();
       StringBuffer sb = new StringBuffer();
       sb.append("<html><head></head>");
       sb.append("<body>Welcome Servlet");
       sb.append("</body></html>");
       writer.print(sb.toString());
   }
}
```


2. 在web.xml中配置xml


```xml

<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                     http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
        version="3.0"
        metadata-complete="true">


   <servlet>
       <servlet-name>WelcomeServlet</servlet-name>
       <servlet-class>andy.com.demo.httpservlet.WelcomeServlet</servlet-class>
       <load-on-startup>20</load-on-startup>
   </servlet>

   <servlet-mapping>
       <servlet-name>WelcomeServlet</servlet-name>
       <url-pattern>/welcome</url-pattern>
   </servlet-mapping>

</web-app>

```

使用web.xml部署应用程序有不少好处:  
 a. 不用重新编译servlet  
 b. 可以使用@WebServlet不能使用的一些配置，比如load-on-startup参数，(配置了load-on-startup就不是在第一次调用servlet时候初始化，而是启动容器后就初始化，这对一些耗时的servlet初始化很有效)



### 4. jsp和MVC
####  1. mvc模式
 mvc模式是将数据和view解耦的一种开啊模式，jsp是一种常用的view实现。
 一般使用servlet作为controller，生成特定的model(数据)，然后根据某些条件，将model于view(jsp)结合，生成用户需要的view。
 
 #### 2. servlet和jsp实现一个mvc模式的例子

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/servlet-mvc.png)

 **controller：**
 ```java
 public class SimpleServletDispatcher extends HttpServlet {

   @Override
   public void doGet(HttpServletRequest req, HttpServletResponse res)
           throws ServletException, IOException {

      //也可以跳到servlet
       RequestDispatcher dispatcher = req.getRequestDispatcher("index1.jsp");

       Map<String,String> map = new HashMap<String,String>();
       map.put("attribute1","attribute1");
       map.put("attribute2","attribute2");

       req.setAttribute("map",map);

       //注意与dispatcher.include(req,res)区别;
       dispatcher.forward(req,res);
   }
}

 ```

 **view(index1.jsp)**
 ```html
 <?xml version="1.0" encoding="ISO-8859-1"?>
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
    <meta http-equiv="content-type" content="text/html; charset=iso-8859-1"/>
</head>
<body>
this is index1.jsp
<%= request.getAttribute("map")%>

</body>
</html>
 ```

 **部署**
 ```xml
 <?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                     http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
        version="3.0"
        metadata-complete="true">

   <servlet>
       <servlet-name>SimpleDispacher</servlet-name>
       <servlet-class>andy.com.demo.httpservlet.SimpleServletDispatcher</servlet-class>
       <load-on-startup>20</load-on-startup>
   </servlet>

   <servlet-mapping>
       <servlet-name>SimpleDispacher</servlet-name>
       <url-pattern>/dispacher</url-pattern>
   </servlet-mapping>
</web-app>

 ```



### 5. Listener
**Listener为Servlet提供了基于事件编程的能力**。

#### 1.  ServletContextListener
ServletContextListener为ServletContext提供了生命周期事件,生命周期函数式**同步调用**。

```java
public class ServletContextListenerImpl implements ServletContextListener {

   public void contextInitialized(ServletContextEvent sce) {
       System.out.println("servlet context Initialized");
       ServletContext context = sce.getServletContext();
       //添加信息
       context.setAttribute("initByContextListener","initByContextListener_");
       System.out.println("## AttributeNames");
       Enumeration<String> ee =  context.getAttributeNames();
       while(ee.hasMoreElements()){
           String name = ee.nextElement();
           System.out.println("##- "+name);
       }
   }

   public void contextDestroyed(ServletContextEvent sce) {
       System.out.println("servlet context destroyed");
   }
}
```
**controller**

```java
public class ListenerServlet extends HttpServlet {

   @Override
   public void doGet(HttpServletRequest req, HttpServletResponse res)
           throws ServletException, IOException {
      //也可以跳到servlet
       RequestDispatcher dispatcher = req.getRequestDispatcher("listenerTest.jsp");

       //注意与dispatcher.include(req,res)区别;
       dispatcher.forward(req,res);
   }
}

```

**view**: listenerTest.jsp
```html
<?xml version="1.0" encoding="ISO-8859-1"?>
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
<head>
   <meta http-equiv="content-type" content="text/html; charset=iso-8859-1"/>
</head>
<body>
this is listenerTest.jsp
<br/>

initByContextListener = <%=application.getAttribute("initByContextListener")%>
<br>
testInitContextParams = <%=application.getInitParameter("testInitContextParams")%>
</body>
</html>
```


**部署web.xml**
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                     http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
        version="3.0"
        metadata-complete="true">

   <servlet>
       <servlet-name>ListenerServlet</servlet-name>
       <servlet-class>andy.com.demo.listener.ListenerServlet</servlet-class>
       <load-on-startup>20</load-on-startup>
   </servlet>

   <servlet-mapping>
       <servlet-name>ListenerServlet</servlet-name>
       <url-pattern>/listenerServlet</url-pattern>
   </servlet-mapping>

   <!-- 初始化ServletContext时候会用到-->
   <context-param>
       <param-name>testInitContextParams</param-name>
       <param-value>testInitContextParams1</param-value>
   </context-param>

   <listener>
       <listener-class>andy.com.demo.listener.ServletContextListenerImpl</listener-class>
   </listener>

</web-app>

```

**结果**  

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/servlet-listener.png)




### 6. filter
#### 1. 主要涉及到3个类
Filter,FilterConfig,FilterChain
```java
public interface Filter {
   public void init(FilterConfig filterConfig) throws ServletException;
   public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain)
           throws IOException, ServletException;
   public void destroy();
}


public interface FilterConfig {
   public String getFilterName();
   public ServletContext getServletContext();
   public String getInitParameter(String name);
   public Enumeration<String> getInitParameterNames();
}
```
####  2. 打印日志的Filter

```java
public class LoggingFilter implements Filter {

   private PrintWriter logger;
   private String prefix;

   public void init(FilterConfig filterConfig) throws ServletException {
       prefix = filterConfig.getInitParameter("prefix");
       String logFileName = filterConfig.getInitParameter("logFileName");
       String appPath = filterConfig.getServletContext().getRealPath("/");
       System.out.println("** appPath = " + appPath);
       System.out.println("** logFileName = " + logFileName);

       try {
           logger = new PrintWriter(new File(appPath, logFileName));
       } catch (Exception e) {
           e.printStackTrace();
           throw new ServletException(e.getMessage(), e.getCause());
       }
   }

   public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

       System.out.println("LoggingFilter.doFilter");

       HttpServletRequest req = (HttpServletRequest) request;
       logger.println(new Date() + " " + prefix + req.getRequestURI());
       logger.flush();
       chain.doFilter(request, response);
   }

   public void destroy() {
       System.out.println("## destroying filter");
       if (logger != null) {
           logger.close();
       }

   }
}

```

web.xml
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                     http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
        version="3.0"
        metadata-complete="true">

...
   <!-- filter -->
   <filter>
       <filter-name>LoggingFilter</filter-name>
       <filter-class>andy.com.demo.filter.LoggingFilter</filter-class>
      
       <init-param>
           <param-name>logFileName</param-name>
           <param-value>log.log</param-value>
       </init-param>
      
       <init-param>
           <param-name>prefix</param-name>
           <param-value>URI:</param-value>
       </init-param>
   </filter>
  
  
   <filter-mapping>
       <filter-name>LoggingFilter</filter-name>
       <url-pattern>/*</url-pattern>
   </filter-mapping>
</web-app>

```

#### 3.多个filter执行顺序
跟配置文件里的顺序一致


### 7. 异步Servlet和Filter
Servlet3.0增加的一个特性，当jsp/servlet需要一个长时间处理的操作时候，他会将哪些操作分配一个新的线程，从而将这个请求处理线程返回到线程池中，服务下一个请求。

实验一下分别在两个线程中执行 asyncServlet
```java
public class AsyncServlet extends HttpServlet {

   @Override
   public void doGet(final HttpServletRequest req, final HttpServletResponse res) throws ServletException, IOException {

       req.setAttribute("threadBefore", Thread.currentThread().getName());
       final AsyncContext context = req.startAsync(req, res);
       context.setTimeout(5000);
       context.start(new Runnable() {
           public void run() {
               try {
                   TimeUnit.SECONDS.sleep(4);
                   req.setAttribute("threadAfter", Thread.currentThread().getName());
                   context.dispatch("/async1.jsp");

               } catch (Exception e) {
                   e.printStackTrace();
               } finally {
                   if (context != null) {
                       context.complete();
                   }
               }
           }
       });
   }
}
```

web.xml

```xml
<web-app>

   <!--async -->
   <servlet>
       <servlet-name>Async</servlet-name>
       <servlet-class>andy.com.demo.async.AsyncServlet</servlet-class>
       <!-- 支持异步 -->
       <async-supported>true</async-supported>
   </servlet>

   <servlet-mapping>
       <servlet-name>Async</servlet-name>
       <url-pattern>/async</url-pattern>
   </servlet-mapping>

</web-app>
```

结果  

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/servlet-async.png)
