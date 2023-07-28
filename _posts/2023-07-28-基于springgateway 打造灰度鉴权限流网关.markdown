---
layout: post
title: 基于springgateway 打造灰度鉴权限流网关
date: 2023-07-28 14:32:00
categories:  开发
---

# 目的
一个小项目经历了快速迭代，逐渐变成了一个较大和较有影响力的西南股，但是一些基本的功能缺失了
    1. 直接上线，没有灰度。
    2. 外部可以调用内部接口，如果被外人抓到，可能会对服务造成影响。
    3. 如果某些接口异常，没有限流功能
   
经过考虑，打算在网关(springgateway)侧补齐这些功能。


# 参考：

https://www.cnblogs.com/huanshilang/p/16023104.html