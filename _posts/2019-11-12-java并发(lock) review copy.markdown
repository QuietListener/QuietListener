---
layout: post
title: java并发(lock) review
date: 2019-10-29 14:32:00
categories:  java review
---

# 锁
## synchronized vs Lock
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


## 读写锁 ReentrantReadWriteLock
  如果只是用lock来保护资源，在**写-写**,**读-写**是适用的，但是只是在**读-读**模式下，加锁是没必要的，特别是读远大于写的情况下。所以使用读写锁会得到更好的并发性能。
  下面的代码模拟读多写少的场景，使用读写锁时候的吞吐量是普通lock的3倍左右~(macPro i5 8G内存)
```java
package andy.com.concurrent.synchronizers;

import org.junit.Test;

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TestReadWriteLock {

    static volatile boolean flag = true;
    static volatile int count = 1;
    static volatile int readActionCount = 1;
    static int countReader = 10;
    static int countWriter = 1;
    static final private CyclicBarrier barrier = new CyclicBarrier(countReader + countWriter);

    public void sleepMillis(int timeMs) {
        try {
            TimeUnit.MILLISECONDS.sleep(timeMs);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 使用普通锁 模拟读多写少的场景
     * 10个读线程线程，读操作耗时20ms
     * 1个写线程，写操作耗时80ms 每隔100毫秒一个写操作
     * 30秒可以并发读2000+次
     *
     * @throws Exception
     */
    @Test
    public void test1() throws Exception {

        final ReentrantLock lock = new ReentrantLock();

        //10个读线程
        for (int i = 0; i < countReader; i++) {
            Thread readThread1 = new Thread() {
                public void run() {
                    try {
                        barrier.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                    System.out.println(this.getName() + " started");
                    while (flag == true) {
                        lock.lock();
                        try {
                            readActionCount += 1;
                            System.out.println(this.getName() + " read:" + count + " readActionCount:" + readActionCount);
                            sleepMillis(20);
                        } finally {
                            lock.unlock();
                        }
                    }
                }
            };

            readThread1.setName("read-" + i);
            readThread1.start();
        }

        //1个写线程
        for (int i = 0; i < countWriter; i++) {
            Thread writeThread1 = new Thread() {
                public void run() {
                    try {
                        barrier.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                    System.out.println(this.getName() + " started");
                    while (flag == true) {
                        lock.lock();
                        try {
                            count += 1;
                            System.out.println(this.getName() + "write:" + count);
                            sleepMillis(80);
                        } finally {
                            lock.unlock();
                        }

                        sleepMillis(100);
                    }
                }
            };

            writeThread1.setName("write-" + i);
            writeThread1.start();
        }


        System.out.println("main started");
        TimeUnit.SECONDS.sleep(30);
        flag = false;
    }


    /**
     * 使用读写说 模拟读多写少的场景
     * 10个读线程线程，读操作耗时20ms
     * 1个写线程，写操作耗时80ms 每隔100毫秒一个写操作
     * 30秒可以并发读7000+次
     *
     * @throws Exception
     */
    @Test
    public void test2() throws Exception {

        final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
        final ReentrantReadWriteLock.ReadLock readLock = lock.readLock();
        final ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock();

        //10个读线程
        for (int i = 0; i < countReader; i++) {
            Thread readThread1 = new Thread() {
                public void run() {
                    try {
                        barrier.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                    while (flag == true) {
                        readLock.lock();
                        try {
                            readActionCount += 1;
                            System.out.println(this.getName() + " read:" + count + " readActionCount:" + readActionCount);
                            sleepMillis(20);

                        } finally {
                            readLock.unlock();
                        }


                    }
                }
            };

            readThread1.setName("read-" + i);
            readThread1.start();
        }

        //1个写线程
        for (int i = 0; i < countWriter; i++) {
            Thread writeThread1 = new Thread() {
                public void run() {
                    try {
                        barrier.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    while (flag == true) {
                        writeLock.lock();
                        try {
                            count += 1;
                            System.out.println(this.getName() + " write:" + count);
                            sleepMillis(80);
                        } finally {
                            writeLock.unlock();
                        }

                        sleepMillis(100);
                    }
                }
            };

            writeThread1.setName("write-" + i);
            writeThread1.start();
        }


        TimeUnit.SECONDS.sleep(30);
        flag = false;
    }
}


```  



# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)
1. [SpringCloud服务的平滑上下线](https://juejin.im/post/5cf63899f265da1b9253c7f4)
