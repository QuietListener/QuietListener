---
layout: post
title:  spring-boot Actuator实现服务平滑下线
date:   2019-10-23 14:32:00
categories:  java spring-boot
---
# 1.背景
> A --调用--> B

以前下线服务都是用的kill，但是由于客户端使用的是robbin+feign进行远程调用，会有缓存。当服务被杀掉的时候，其实还是有请求会发到服务端。
比如:上面 A调用B，我们kill掉B服务后，A重服务中心(Eureka)感知到B下线最长需要30s，加上A使用robin+feign会缓存B点信息，所以可能在杀掉B后的几十秒后依然有请求发送到B。这样下线非常暴力。

# 2.目标
 当B下线后，通知注册中心快速感知B下线，当A没有http请求到B的时候再杀掉B。

# 3.spring-boot 的 Actuator

# 参考
1. [分布式事务-本地消息表：最终一致性](https://quguang.wang/post/transaction-local-msg-tb/)


