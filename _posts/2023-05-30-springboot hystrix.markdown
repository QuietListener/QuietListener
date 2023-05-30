---
layout: post
title: springboot hystrix
date: 2023-05-27 14:32:00
categories:  开发
---

springboot使用hystrix的熔断降级功能
1. 添加 @EnableCircuitBreaker
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
@ComponentScan(basePackages = {"com.andy.test.common.gray","com.andy.test.api"})
@EnableFeignClients(basePackages = {"com.andy.test.api.remote" })//指定feignclient所在的包
public class ApiApplication {
    public static void main(String [] args)
    {
        SpringApplication.run(ApiApplication.class, args);
    }
}

```

2. 在需要熔断的service上使用
   ```java
    @HystrixCommand(
            commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")})
    public String call(String name){
        return backendApi.hello(name);
    }
   ```