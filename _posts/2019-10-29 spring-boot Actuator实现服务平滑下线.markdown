---
layout: post
title:  spring-boot Actuator实现服务平滑下线
date:   2019-10-29 14:32:00
categories:  java spring-boot
---
# 1.背景
> A --调用--> B

以前下线服务都是用的kill，但是由于客户端使用的是robbin+feign进行远程调用，会有缓存。当服务被杀掉的时候，其实还是有请求会发到服务端。
比如:上面 A调用B，我们kill掉B服务后，A重服务中心(Eureka)感知到B下线最长需要30s，加上A使用robin+feign会缓存B点信息，所以可能在杀掉B后的几十秒后依然有请求发送到B。这样下线非常暴力。

# 2.目标
 当B下线后，通知注册中心快速感知B下线，当A没有http请求到B的时候再杀掉B。

# 3.spring-boot 的 Actuator
Spring boot actuator 提供了很多的 函数可以查看机器，服务状态，和一些接口来控制服务。
其中有一个service-register接口，可以控制当前服务在“服务中心”(eureka) 注册或者注销的操作。
如下：
## 1.开启 service-register接口
#### 1. pom中加入
```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

#### 2. 在application-xxx.yml中添加
```xml
management:  #actuator
  server:
    port: 6082
  endpoints:
    web:
      base-path: /
      exposure:
        include: "*"
```

#### 3. register or un-register
```shell
xxx git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=UP"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"  
xxx git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=DOWN"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
```
可以register或者un-register这台机器
 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20191029-springboot-acurator.jpg)

## 2.下线服务过程
```
air-wallpaper git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
air-wallpaper git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=CANCEL_OVERRIDE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
```

# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)


