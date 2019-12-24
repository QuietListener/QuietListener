---
layout: post
title: spring-cloud springboot
date: 2019-12-24 14:32:00
categories:  java javascript
---
spring-boot的好东西
# 1. 一个趋势
以前java Web服务很多都是将程序代码打包为jar，然后将部署到web容器，tomcat ，jetty等。现在spring-boot刚好反过来，将web容器(tomcat,jetty等) 打包进jar。 这样可以像执行一个普通jar包一样来发布web应用，也省去了以前那种每台机器都去配置一个tomcat的麻烦。

# 1. spring-boot好在哪里?
主要有三个方面
## 1. 自动配置
   常见的应用功能，spring自动提供相关配置
2. 起步依赖 starter
   告诉spring-boot需要什么功能，它就引入需要的库，并且这些库是经过spring-boot 开发团队测试过的，不会有问题。不用像以前一样配一个spring-mvc 满世界找依赖，找到了以后还可能有版本不兼容问题。
3. acuator
   了解spring运行的情况。

# 2. 怎么实现的。