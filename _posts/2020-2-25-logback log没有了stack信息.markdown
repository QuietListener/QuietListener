---
layout: post
title: logback log没有了stack信息
date: 2020-2-25 14:32:00
categories:  java  logback
---

# 1. 背景
 受冠状病毒疫情影响，学生都在家远程上课，日活屡创新高。 客服那边报过来的问题多了很多。但是在检查对战的时候发现，只有Exception的message，没有了堆栈信息，这样不容易定位问题。
 如下所示: 只显示了一个 **java.lang.NullPointerException: null**

```log
__source__:  10.1.7.7
__tag__:__hostname__:  xxx
__tag__:__path__:  /var/www/data/log/xxx/api/api.log
__topic__:  
application_name:  xxx-api
extra:  {[null,0,6],EMPTY,[{"name":"access_token","value":"xxx","version":0,"comment":null,"domain":null,"maxAge":-1,"path":null,"secure":false,"httpOnly":false}]}
java.lang.NullPointerException: null
level:  ERROR
method:  method3
parent_span_id:  xxxx
server_ip:  10.1.7.7
span_id:  xxx
status:  0
thread:  http-nio-9000-exec-133
time:  2020-02-19 20:09:04
time_consume:  42
trace_id:  xxx
user_token:  xxx/xxx
```

2. 原因
hotspot虚拟机发现某个Exception平凡抛出的时候，虚拟机会重新编译，并优化，抛出的异常不会提供 stack trace。


> The compiler in the server VM now provides correct stack backtraces for all "cold" built-in exceptions. For performance purposes, when such an exception is thrown a few times, the method may be recompiled. After recompilation, the compiler may choose a faster tactic using preallocated exceptions that do not provide a stack trace. To disable completely the use of preallocated exceptions, use this new flag: -XX:-OmitStackTraceInFastThrow.


3. 解决使用 
像文档里提到的一样，添加一个参数禁用这个功能: -XX:-OmitStackTraceInFastThrow 
