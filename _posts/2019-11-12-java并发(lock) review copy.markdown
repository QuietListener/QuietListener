---
layout: post
title: java并发(lock) review
date: 2019-10-29 14:32:00
categories:  java review
---

# 锁
1.锁是解决线程安全问题的一个办法，(还可以使用 ThreadLocal,不变对象等方法)
java中锁的实现:synchronized和Lock

1. synchronized
  Java中最基本的互斥同步手段是synchronized关键字。synchronized编译为字节码后会在同步块前后分别形成monitorenter和monitorexit这两个字节码指令，这两个字节码都需要一个reference指定要锁定和解锁的对象。在monitorenter指令执行时，首先会去尝试获取对象的锁，如果这个对象没有被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加1。在执行monitorexit指令时，会将锁的计数器减1. 所以可以看出synchronized是可重入的。
2. ReentrantLock
java.util.concurrent包提供了ReentranLock来提供同步，和synchronized相同，一个在语法层面提供，一个在api层面提供（使用lock，unlock配合try-finally来实现）。
Reentranlock提供了一些高级的功能。
1.等待可中断
   当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待。
2.公平所
  多个线程等待同一个锁的时候，必须按照申请锁的时间顺序来依次获得锁（Reentranlock和和synchronized都是非公平锁）
3.ReentrantLock可以同时绑定多个Condition对象
ReentrantLock的wait notify或者notifyAll可以实现一个隐含条件，




# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)
1. [SpringCloud服务的平滑上下线](https://juejin.im/post/5cf63899f265da1b9253c7f4)
