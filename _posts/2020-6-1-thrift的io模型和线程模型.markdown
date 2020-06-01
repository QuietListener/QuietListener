---
layout: post
title: thrift的io模型和线程模型
date: 2020-5-8 14:32:00
categories:  java maven
---

# 1. why thrift
公司最早开始使用thrift是在2014年，在高速发在期间， 迭代非常快。 当时为了减少沟通成本，加快迭代速度。考虑了技术成熟度(facebook出品)，决定使用thrift开发。

# 2.io模型和线程模型
公司使用的是TThreadedSelectorServer，在单机高并发方面还是非常给力。
这里总结一下thrift的io模型和线程模型，这里用io来划分，分为NIO和BIO两部分

## 1. BIO（阻塞io）
1. TSimpleServer 
读写使用阻塞io，accept/read/write和业务逻辑都在一个线程里完成。多用来测试

2. TThreadPoolServer 
读写使用阻塞io，accept在一个单独线程，每建立一个链接，将所有的read/write 和业务逻辑放到一个线程池处理。

## 2. NIO
1. TNonblockingServer 
使用非阻塞io, 单个selector， select/accept/read/write和业务逻辑都在一个线程中处理。

2. THsHaServer 
使用非阻塞io, 单个selector， select/accept/read/write在一个线程，业务逻辑放入到一个线程池处理。
HsHa = Half Sync/Half Async ，我自己理解为read，write，accept这些是同步，业务处理是异步，所以是半同步半异步。

3. TThreadedSelectorServer 
使用非阻塞io, 一个selector在AcceptThread线程中处理accept事件，然后多个SelectorThread线程(每个线程一个selector,名字叫)处理read/write事件，业务逻辑放入到一个线程池处理(WorkerThread)。
这样更好的利用了多核cpu的能力，获取更好的并发性,也是百词斩线上使用的Server。


# 3. 这几个server的继承关系如下图

![thrift]](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-06-01-thrift-io-thread.jpg)

	
