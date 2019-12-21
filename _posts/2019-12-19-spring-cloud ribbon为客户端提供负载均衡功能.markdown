---
layout: post
title: spring-cloud ribbon为客户端提供负载均衡功能
date: 2019-12-19 14:32:00
categories:  java javascript
---
# 1. 写在前面
公司项目 使用spring-cloud 微服务架构， 使用ribbon来做client端的负载均衡。在spring-cloud里面基本只需要一个配置就可以了，但是还是看看ribbon是个什么东西.

# 2. what 
## 1. ribbon是什么？
Ribbon是一个远程调用库，他提供了负载均衡的功能。
> Ribbon is a Inter Process Communication (remote procedure calls) library with built in software load balancers.

## 2. ribbon 提供了哪些feature?
>Load balancing  负载均衡
>Fault tolerance 容错
>Multiple protocol (HTTP, TCP, UDP) support in an asynchronous and reactive model 以异步和响应式的方式支持http，udp ,udp
>Caching and batching 缓存和批处理


## 3 demo
### 1. 配置文件 p1.properties
```shell
#Client的className,可以指定自定义的Client
sample-client.robbin.ClientClassName=com.netflix.niws.client.http.RestClient

# Max number of retries on the same server (excluding the first try)
#在选择的机器上最多尝试次数
sample-client.ribbon.MaxAutoRetries=0

# Max number of next servers to retry (excluding the first server)
# 如果在选择的机器上挂了，还可以选几台机器试一试
sample-client.ribbon.MaxAutoRetriesNextServer=2

# Whether all operations can be retried for this client
sample-client.ribbon.OkToRetryOnAllOperations=true

# Interval to refresh the server list from the source
#刷新server列表的时间间隔
sample-client.ribbon.ServerListRefreshInterval=2000

# Connect timeout used by Apache HttpClient
sample-client.ribbon.ConnectTimeout=3000

# Read timeout used by Apache HttpClient
sample-client.ribbon.ReadTimeout=3000

# Initial list of servers, can be changed via Archaius dynamic property at runtime
# server 列表
sample-client.ribbon.listOfServers=www.microsoft.com:80,www.yahoo.com:80

#sample-client.ribbon.EnablePrimeConnections=true
```
### 2. java代码
```java
import com.netflix.client.ClientFactory;
import com.netflix.client.http.HttpRequest;
import com.netflix.client.http.HttpResponse;
import com.netflix.config.ConfigurationManager;
import com.netflix.loadbalancer.ZoneAwareLoadBalancer;
import com.netflix.niws.client.http.RestClient;
import org.junit.Test;

import java.net.URI;

public class RibboTest1 {

    @Test
    public void test1() throws Exception {

        //初始化
        String keyServers = "sample-client.ribbon.listOfServers";
        ConfigurationManager.loadPropertiesFromResources("p1.properties");  // 1


        System.out.println("server list:"+ConfigurationManager.getConfigInstance().getProperty(keyServers));
        RestClient client = (RestClient) ClientFactory.getNamedClient("sample-client"); // 2
        HttpRequest request = HttpRequest.newBuilder().uri(new URI("/")).build(); // 3
        for (int i = 0; i < 6; i++) {
            HttpResponse response = client.executeWithLoadBalancer(request); // 4
            System.out.println(i + ": Status code for " + response.getRequestedURI() + "  :" + response.getStatus());
        }

        /**
         * 打印loadbalancer的信息
         */
        System.out.println("loadbalancer 信息:");
        ZoneAwareLoadBalancer lb = (ZoneAwareLoadBalancer) client.getLoadBalancer();
        System.out.println(lb.getLoadBalancerStats());


        /**
         * 刷新server列表
         */
        ConfigurationManager.getConfigInstance().setProperty(
                keyServers, "www.linkedin.com:80,www.microsoft.com:80"); // 5
        System.out.println("server list:"+ConfigurationManager.getConfigInstance().getProperty(keyServers));
        System.out.println("changing servers ...");

        Thread.sleep(3000); // 6
        for (int i = 0; i < 6; i++) {
            HttpResponse response = null;
            try {
                response = client.executeWithLoadBalancer(request);
                System.out.println(i + ": Status code for " + response.getRequestedURI() + "  :" + response.getStatus());
            } finally {
                if (response != null) {
                    response.close();
                }
            }
        }

        /**
         * 打印loadbalancer的信息
         */
        System.out.println("loadbalancer 信息:");
        System.out.println(lb.getLoadBalancerStats()); // 7
    }
}

```

### 3.运行结果
```java
server list:[www.microsoft.com:80, www.yahoo.com:80]
0: Status code for http://www.yahoo.com:80/  :301
1: Status code for http://www.microsoft.com:80/  :302
2: Status code for http://www.yahoo.com:80/  :301
3: Status code for http://www.microsoft.com:80/  :302
4: Status code for http://www.yahoo.com:80/  :301
5: Status code for http://www.microsoft.com:80/  :302
loadbalancer 信息:
Zone stats: {unknown=[Zone:unknown;	Instance count:2;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
},Server stats: [[Server:www.microsoft.com:80;	Zone:UNKNOWN;	Total Requests:3;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Sat Dec 21 16:20:37 CST 2019;	First connection made: Sat Dec 21 16:20:33 CST 2019;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:33.666666666666664;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:27.0;	max resp time:47.0;	stddev resp time:9.428090415820636]
, [Server:www.yahoo.com:80;	Zone:UNKNOWN;	Total Requests:3;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Sat Dec 21 16:20:34 CST 2019;	First connection made: Sat Dec 21 16:20:32 CST 2019;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:1519.6666666666667;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:428.0;	max resp time:3454.0;	stddev resp time:1371.55248613468]
]
server list:[www.linkedin.com:80, www.microsoft.com:80]
changing servers ...
0: Status code for http://www.microsoft.com:80/  :302
1: Status code for http://www.linkedin.com:80/  :404
2: Status code for http://www.microsoft.com:80/  :302
3: Status code for http://www.linkedin.com:80/  :404
4: Status code for http://www.microsoft.com:80/  :302
5: Status code for http://www.linkedin.com:80/  :404
loadbalancer 信息:
Zone stats: {unknown=[Zone:unknown;	Instance count:2;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
},Server stats: [[Server:www.microsoft.com:80;	Zone:UNKNOWN;	Total Requests:6;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Sat Dec 21 16:20:41 CST 2019;	First connection made: Sat Dec 21 16:20:33 CST 2019;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:30.833333333333332;	90 percentile resp time:28.0;	95 percentile resp time:28.0;	min resp time:27.0;	max resp time:47.0;	stddev resp time:7.266743118863888]
, [Server:www.yahoo.com:80;	Zone:UNKNOWN;	Total Requests:3;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Sat Dec 21 16:20:34 CST 2019;	First connection made: Sat Dec 21 16:20:32 CST 2019;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:1519.6666666666667;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:428.0;	max resp time:3454.0;	stddev resp time:1371.55248613468]
, [Server:www.linkedin.com:80;	Zone:UNKNOWN;	Total Requests:3;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Sat Dec 21 16:20:41 CST 2019;	First connection made: Sat Dec 21 16:20:40 CST 2019;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:150.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:126.0;	max resp time:195.0;	stddev resp time:31.843366656181317]
]
Disconnected from the target VM, address: '127.0.0.1:57450', transport: 'socket'
```

### 4.项目地址
[robbin httpclient demo ](https://github.com/QuietListener/ribbon/blob/master/src/main/java/ribbon/RibboTest1.java)
