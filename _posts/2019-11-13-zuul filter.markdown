---
layout: post
title: zuul filter
date: 2019-10-29 14:32:00
categories:  spring-cloud zuul
---
# 什么是ZuulFilter
zuul 作为网关，提供了ZuulFilter机制，来扩展它的功能。可以通过ZuulFilter来实现过滤,鉴权,log等功能。
ZuulFilter 主要有3类
1. PRE Filters 在请求后端服务器之前调用
2. ROUTING Filter 实现请求转发，并获取结果
3. POST Filter  获得请求结果之后调用
4. ERROR Filter 如果上面任何一个阶段发生错误，会调用ERROR Filter

系统默认实现了很多filter在org.springframework.cloud.netflix.zuul.filters里面，需要了解的可以看源码。如果自己需要，可以定制自己的Filter。每个自定义的Filter需要实现 Zuul的 filterType,filterOrder,shouldFilter,run 这方法。各个Filter可以通过一个RequestContext(ThreadLocal实现)来交换数据。
``` java

    /**
    *  filter的类型
    */
    @Override
    public String filterType() { return "pre";}

    /**
    *  order 顺序，越小优先级越高
    */
    @Override
    public int filterOrder() {  return -1;  }

    /**
    *  是否需要执行
    */
    @Override
    public boolean shouldFilter() {return true;}

    /**
    * 具体逻辑写在这里
    */
    @Override
    public Object run() throws ZuulException { 

         // RequestContext用来Filter之间数据共享，本质上是一个ThreadLocal
         RequestContext ctx = RequestContext.getCurrentContext();
         ....
    }

```
# 生命周期
下面是ZuulFilter的生命周期图:

![](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)



# 例子: 实现记录每个请求耗时的ZuulFilter。
需要两个Filter：
1. PreTimeFilter 记录请求开始的时间和url，并存放在 RequestContext中
2. PostLogFilter 从RequestContext中取出url和请求开始时间，计算耗时，写日志

```java
/**
* 记录请求开始的时间和url，并存放在 RequestContext中
*/
public class PreTimeFilter extends ZuulFilter {

    static public final String START_TIME_MS = "start_time_ms";
    static public final String REQUEST_URL = "my_request_url_";

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return -1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {

        try {
            RequestContext ctx = RequestContext.getCurrentContext();
            ctx.put(START_TIME_MS,System.currentTimeMillis());
            ctx.put(REQUEST_URL,ctx.getRequest().getRequestURI());
        } catch (Exception e) {
        }

        return null;
    }
}

```

```java

/**
* 从 RequestContext中取出url和请求开始时间，并写日志
*/
public class PostLogFilter extends ZuulFilter {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        return 2000;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {

        try {
            RequestContext ctx = RequestContext.getCurrentContext();

            String url = (String)ctx.get(PreTimeFilter.REQUEST_URL);
            int httpStatus = ctx.getResponseStatusCode();
            long startMs = (Long) ctx.get(PreTimeFilter.START_TIME_MS);
            long cost = System.currentTimeMillis() - startMs;

            String serviceId = (String) ctx.get(FilterConstants.SERVICE_ID_KEY);
            ZuulLogger.log(url, serviceId, httpStatus, cost);
        } catch (Exception e) {
        }

        return null;
    }
}

```
# 参考
1. [How ZuulFilter works](https://github.com/Netflix/zuul/wiki/How-it-Works)
