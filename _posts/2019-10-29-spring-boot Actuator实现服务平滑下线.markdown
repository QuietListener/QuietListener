---
layout: post
title: spring-boot Actuator实现服务平滑下线
date: 2019-10-29 14:32:00
categories:  java spring-boot
---

# 1.背景
> A --调用--> B

以前下线服务都是用的kill，但是由于客户端使用的是robbin+feign进行远程调用，会有缓存。当服务被杀掉的时候，其实还是有请求会发到服务端。
比如:上面 A调用B，我们kill掉B服务后，A重服务中心(Eureka)感知到B下线最长需要30s，加上A使用robin+feign会缓存B点信息，所以可能在杀掉B后的几十秒后依然有请求发送到B。这样下线非常暴力。

# 2.目标
 当B下线后，通知注册中心(我们用的eureka)快速感知B下线，没有请求到B的时候再杀掉B。这样平滑下线。

# 3. 实现: spring-boot 的 Actuator
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

#### 3. register 和 un-register 接口
```shell
xxx git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=UP"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"  
xxx git:(master) ✗ curl -X "POST" "http://localhost:6082/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
```
可以register或者un-register这台机器
 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20191029-springboot-acurator.jpg)

#### 4. overriddenStatus 和 status

```shell
shell git:(master) ✗ curl -X "GET" "http://localhost:6081/service-registry"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
{"overriddenStatus":"UNKNOWN","status":"UP"}

```
我们看到有两个状态overriddenStatus和status。overriddenStatus是覆盖状态。   
服务的最终状态由overriddenStatus和status共同决定，他的工作逻辑如下:    
**如果overriddenStatus不为UNKONWN，最终状态就用overriddenStatus，**   
**如果overriddenStatus为UNKONWN, 就用status的状态。** 

例如:
当我们设置服务状态为OUT_OF_SERVICE时候, overriddenStatus和status都为OUT_OF_SERVICE。    
这时候服务再向eureka server发心跳，eureka server依然会认为服务是OUT_OF_SERVICE,因为overriddenStatus不为UNKNOW。(开始我不知道overriddenStatus的作用，以为状态会变成UP状态)    

**为什么需要有  overriddenStatus ?**   
  试想一下假如没有overriddenStatus，只有status一个状态。 我们调用 curl -X "POST" "http://localhost:6082/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"成功,此时status = OUT_OF_SERVICE。服务标记为已经下线。 这时候我们的服务还没有停止，还在向eureka server发送心跳，这时候服务的状态又被改回了UP状态。 所以一个状态不够。


## 2.下线,上线服务过程
"POST" "http://localhost:6082/service-registry?status=OUT_OF_SERVICE" 下线服务 将overriddenStatus和status设置为OUT_OF_SERVICE   
"POST" "http://localhost:6082/service-registry?status=CANCEL_OVERRIDE" 将overriddenStatus和status设置为unkonw

下线:
1. 调用 POST "http://localhost:6082/service-registry?status=OUT_OF_SERVICE" 下线服务 overriddenStatus和status设置为OUT_OF_SERVICE。
2. shell脚本监控接口日志，等到日志没有更新，说明没有流量打到这个服务了。
3. kill掉服务。这做到了平滑下线。

stop.sh 脚本
```shell
#!/bin/bash

pid="../pid"
watch_log="../logs/biz1.log"
last_line=""
previous_line=""
if [ -f $pid ]; then
    `rm _previous.txt`
fi

#下线服务 可以多掉几次保险起见
j=0
while [ $j -le 30 ]
do
    `curl -X "POST" "http://localhost:6082/actuator/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`
    ret="`curl -X "GET" "http://localhost:6082/actuator/service-registry"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`"
    echo  "$j ret: ${ret}"

    match=`echo $ret | grep "OUT_OF_SERVICE"`
    echo "match: $match ${#match}"
    if [ ${#match} -gt 5 ]; then #如果match字符串不为空 跳出循环
        break;
    fi

    let j++;
    sleep 0.2
done



#下面检测log是否还在更新，最多等30秒
i=0
match_count=0;
while [ $i -le 30 ]
do
    last_line="`tail -n 1 $watch_log`"
    previous_line="`tail -n 1 _previous.txt`"
    #echo ${last_line}
    #echo ${previous_line}

    #如果没有更新说明没有流量到这个服务了
    if [ "$last_line" = "$previous_line" ]; then
        echo "${i} `date +%Y_%m_%d%t%H:%M:%S` eq"
        let match_count++;
        #echo "match_count: $match_count";

         #如果有2次检查都没有变化才退出
        if [ $match_count -gt 1 ]; then
             break
        fi
    else
        echo "${i} `date +%Y_%m_%d%t%H:%M:%S` not eq"
    fi

    sleep 1
    let i++
    `echo $last_line > _previous.txt`
done


#杀掉服务
if [ -f $pid ]; then
    if kill `cat $pid` ; then
        echo 'kill process succeed'
    else
        echo 'kill process fail'
        exit 1
    fi
else
    echo 'pid file not exists'
    exit 1
fi

```

上线:
1. 启动服务
2. 调用 POST "http://localhost:6082/service-registry?status=CANCEL_OVERRIDE" 下线服务 overriddenStatus和status设置为UNKNOW
3. 调用 POST "http://localhost:6082/service-registry?status=UP" 下线服务 overriddenStatus和status设置为UP


# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)
1. [SpringCloud服务的平滑上下线](https://juejin.im/post/5cf63899f265da1b9253c7f4)
