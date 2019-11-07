---
layout: post
title: spring-boot Actuator实现服务平滑下线
date: 2019-10-29 14:32:00
categories:  java spring-boot
---

# 1.背景
> 服务A --调用-->  服务B

以前下线服务都是用的kill，但是由于服务A(spring-boot)使用的是robbin+feign进行远程调用，会有缓存（默认都是30s）。当服务B被杀掉的时候，A需要等待一段很长的时间才会感知到B下线，所以尽管B已经被杀掉了，还是有A的请求发到服务B。非常暴力

# 2.目标
当B下线后，通知注册中心(我们用的eureka)快速感知B下线，当没有请求到B的时候再杀掉B。这样平滑下线。

**如果只想知道How不想知道why,可以直接跳到:4.总结**
# 3. 实现: 
## spring-boot 的 Actuator
Spring boot actuator 提供了很多的 函数可以查看机器，服务状态，和一些接口来控制服务。
其中有一个service-register接口，可以控制当前服务在“服务中心”(eureka) 注册或者注销的操作。
如下：
## 1.开启 service-register接口
#### 1. pom中加入
```xml
       <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
                <version>2.0.3.RELEASE</version>
            </dependency>
```

#### 2. 在application-xxx.yml中添加
```yml
management:  #actuator
  server:
    port: 12345
  endpoints:
    web:
      #base-path: /
      exposure:
        include: "service-registry"
```

#### 3. service-registry 接口 可以修改服务的状态
例如下面两次调用将服务状态设置为UP或者OUT_OF_SERVICE
```shell
xxx git:(master) ✗ curl -X "POST" "http://localhost:12345/service-registry?status=UP"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"  
xxx git:(master) ✗ curl -X "POST" "http://localhost:12345/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
```
可以register或者un-register这台机器
 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20191029-springboot-acurator.jpg)

#### 4. 每个服务有的状态由overriddenStatus和status共同决定

下面是使用 GET 获取某服务的状态
```shell
shell git:(master) ✗ curl -X "GET" "http://localhost:6081/service-registry"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"
{"overriddenStatus":"UNKNOWN","status":"UP"}

```
我们看到每个服务有两个状态overriddenStatus和status。overriddenStatus是覆盖状态。   
服务的最终状态由overriddenStatus和status共同决定，他的工作逻辑如下:    
**如果overriddenStatus不为UNKONWN，最终状态就用overriddenStatus，**   
**如果overriddenStatus为UNKONWN, 就用status的状态。** 

例如:
当我们设置服务状态为OUT_OF_SERVICE时候, overriddenStatus和status都为OUT_OF_SERVICE。    
这时候服务再向eureka server发心跳，eureka server依然会认为服务是OUT_OF_SERVICE,因为overriddenStatus不为UNKNOWN。(开始我不知道overriddenStatus的作用，以为状态会变成UP状态)    

**为什么需要有  overriddenStatus ?**   
  试想一下假如没有overriddenStatus，只有status一个状态。 我们调用 curl -X "POST" "http://localhost:12345/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"成功,此时status = OUT_OF_SERVICE。服务标记为已经下线。 这时候我们的服务还没有停止，还在向eureka server发送心跳，这时候服务的状态又被改回了UP状态。 所以一个状态不够。


## 2.实现平滑下线服务过程
下面是需要用到的两个请求
>"POST" "http://localhost:12345/service-registry?status=OUT_OF_SERVICE" 下线服务 将overriddenStatus和status设置为OUT_OF_SERVICE   
>"POST" "http://localhost:12345/service-registry?status=CANCEL_OVERRIDE" 将overriddenStatus和status设置为unkonw

**下线过程:**
1. 调用 POST "http://localhost:12345/service-registry?status=OUT_OF_SERVICE" 下线服务 overriddenStatus和status设置为OUT_OF_SERVICE。
2. shell脚本监控接口日志，等到日志没有更新，说明没有流量打到这个服务了。
3. 调用 POST "http://localhost:12345/service-registry?status=CANCEL_OVERRIDE" 下线服务 overriddenStatus和status设置为UNKNOWN。
4. kill掉服务。这做到了平滑下线。

上面的逻辑写到了 stop.sh 脚本了，如下(其中watch_log需要修改为具体服务的log地址):
```shell
#!/bin/bash

# 待监控的log路径 根据项目情况自己添加
watch_log="./logs/rpc.log"

# ========需要添加的部分===========
last_line=""
previous_line=""
if [ -f "_previous.txt" ]; then
    `rm _previous.txt`
fi

echo "### 1. get the service offline  ###"
j=0
while [ $j -le 30 ]
do
    `curl -X "POST" "http://localhost:12345/actuator/service-registry?status=OUT_OF_SERVICE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`
    ret="`curl -X "GET" "http://localhost:12345/actuator/service-registry"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`"
    echo  "$j ret: ${ret}"

    match=`echo $ret | grep "OUT_OF_SERVICE"`
    echo "match: $match ${#match}"
    if [ ${#match} -gt 5 ]; then #如果match字符串不为空 跳出循环
        break;
    fi

    let j++;
    sleep 0.2
done


echo "### the service is offline now  ###"
echo "";
echo "### 2. monitor log: ${watch_log}  ###"

#下面检测log是否还在更新，最多等30秒
i=0
match_count=0;
while [ $i -le 60 ]
do
    last_line="`tail -n 1 $watch_log`"
    previous_line="`tail -n 1 _previous.txt`"
    #echo ${last_line}
    #echo ${previous_line}

    #如果没有更新说明没有流量到这个服务了
    if [ "$last_line" = "$previous_line" ]; then
        echo "${i} `date +%Y_%m_%d%t%H:%M:%S` ${watch_log} stop changing"
        let match_count++;
        #echo "match_count: $match_count";

         #如果有2次检查都没有变化才退出
        if [ $match_count -gt 1 ]; then
             break
        fi
    else
        echo "${i} `date +%Y_%m_%d%t%H:%M:%S` ${watch_log} keep changing"
    fi

    sleep 1
    let i++
    `echo $last_line > _previous.txt`
done

echo "### no service is calling this service now ###"
echo "";
echo "### 3. reset the override status ###"

k=0;
while [ $k -le 30 ]
do
    `curl -X "POST" "http://localhost:12345/actuator/service-registry?status=CANCEL_OVERRIDE"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`
    ret="`curl -X "GET" "http://localhost:12345/actuator/service-registry"  -H "Content-Type: application/vnd.spring-boot.actuator.v2+json;charset=UTF-8"`"
    echo  "$k ret: ${ret}"

    match=`echo $ret | grep "UNKNOWN"`
    echo "match: $match ${#match}"
    if [ ${#match} -gt 5 ]; then #如果match字符串不为空 跳出循环
        break;
    fi

    let k++;
    sleep 0.2
done

echo "### finished reset the override status  ###"

echo "";
echo "### 4. kill the service  ###"

# ========需要添加的部分 end===========


# ======== 下面是原来的下线逻辑 ===========
if [ -f pid ]; then
    if kill `cat pid` ; then
        echo 'kill process succeed'
    else
        echo 'kill process fail'
        exit 1
    fi
else
    echo 'pid file not exists'
    exit 1
fi


if [ -f "_previous.txt" ]; then
    `rm _previous.txt`
fi

```

## 3.减小感知到服务下线的时间
下面是spring-cloud远程方法调用的机制:
1. spring-cloud 使用 feign实现远程方法调用。
2. feign使用robbin在客户端实现负载均衡。
3. robbin负载均衡需要从Eureka Client获取各个服务的信息(ip,端口,状态等)
4. Eureka Client 从 Eureka Service 拉取各个服务信息(ip,端口,状态等)。

**Eureka Client 会定期从Eureka Service 拉取服务信息,这个时间间隔由下面这个参数控制（默认为30秒）**
```xml
eureka:
  client:
    registry-fetch-interval-seconds: 5 #拉取其他服务注册信息频率 
```    

**Robbin 会定期从 Eureka Client 拉取服务信息并缓存。这个时间由下面这个参数决定（默认为30秒）**
```xml
ribbon:
  ServerListRefreshInterval: 5000
```
**所以叫加快服务感知到到其他服务下线的时间 需要减小上面这两个参数。我这里修改为5秒**


# 4.总结
需要做的事:
1. 修改 pom.xml 添加依赖
```xml
       <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
                <version>2.0.3.RELEASE</version>
            </dependency>
```


2. 修改 application-xxx.xml 

```yml
##修改从Eureka Service 拉取服务状态信息的频率为5秒，默认为30秒
eureka:
  client:
    registry-fetch-interval-seconds: 5 #拉取其他服务注册信息频率 

##robbin 缓存刷新时间改为5秒 默认为30秒
ribbon:
  ServerListRefreshInterval: 5000

management:  #actuator
  server:
    port: 12345
  endpoints:
    web:
      #base-path: /
      exposure:
        include: "service-registry"
```

3. 替换stop脚本为上面的stop脚本并修改 watch_log为需要的log
```shell   
    # 待监控的log路径 根据项目情况自己添加
    watch_log="./logs/rpc.log"
```

# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)
2. [SpringCloud服务的平滑上下线](https://juejin.im/post/5cf63899f265da1b9253c7f4)
