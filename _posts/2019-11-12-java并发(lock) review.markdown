---
layout: post
title: java并发(lock) review
date: 2019-10-29 14:32:00
categories:  java review
---

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


## 自建同步器
### 要解决的问题，状态管理
   在单线程中如果程序依赖一个先验条件，无法被满足了，那么它永远都不会变为真了。但是在**多线程环境下**就不一样了，这个先验条件有可能在其他线程中被改变。例如BlockQueue 当queue为空的时候，去读取会阻塞，当队列满时候去写入也会阻塞。
   解决这个问题有几种方式，比较差的方式有**轮询等待** **轮询+随眠**。轮询等待就是在一个循环不断去检测是否条件为真，这样很消耗cpu； **轮询+随眠**在一个循环不断去检测是否条件为真，当伪假时候睡眠一段时间再检查，这样cpu消耗变少了，但是也睡过了头。

### 比较好的解决办法 条件队列
条件队列可以让一组线程---等待集---以某种方式等待相关条件变真。条件队列里面存放的不是数据，是线程。
**java中任意对象都可以是锁，也可以是一个条件队列。**

**锁,条件谓词,条件队列**
条件谓词就是依赖的变量的某种状态 比如:queue为空
使用条件队列关键在与识别对象可以等待的**条件谓词**  
下面是一个条件队列实现的可关闭的锁的例子。

 ```java
 package andy.com.concurrent.synchronizers.conditionQueue;

/**
 * 条件队列例子:可以关闭的闸门。
 *
 * 三元关系:条件队列 条件谓词 锁
 *
 * 为什么要加一个 generation？
 * 试想如果不加 generation，当砸门打开再快速关闭，await只检查isOpen,某些线程可能永远都只会在
 *  while (isOpen == false)
 *     await()
 * 中，不会被唤醒了
 */
public class TheadGage {

    //条件谓词就是: (isopen || generation > n)
    private boolean isOpen = false;
    private int generation;


    public synchronized void close() {
        isOpen = false;
    }

    public synchronized void open() {
        ++generation;
        isOpen = true;
        notifyAll();
    }

    /**
     * 放开闸门的条件 : isOpen == true || arrivalGeneration != generation
     * @throws InterruptedException
     */
    public synchronized void await() throws InterruptedException {
        int arrivalGeneration = generation;
        while (isOpen == false && arrivalGeneration == generation) {
            wait();
        }
    }


    public static void main(String [] args) throws InterruptedException{

    }
}


 ```    


### 显示锁，Lock和Condition 
#### 为什么需要Condition?
Condition是广义的条件队列，内部条件队列，一个锁只能对应一个条件队列，这样就很低效了。比如生产者-消费者模型。 
如果只有一条件队列，生产者和消费者线程都在这个队列中，当数据队列不为空的时候，应该唤醒消费者线程，但是由于只有一个条件队列，所以当执行notifyAll的时候会唤醒条件队列中所有线程，其实把生产这线程也唤醒了。 这样会带来不必要的上下文切换，浪费cpu资源。   
所以如果有两个条件队列:消费者队列，生产者队列。当数据队列不为空的时候只唤醒消费者队列中的线程，当数据不满的时候唤醒生产者队列中的线程就会减少不必要的上下文切换。
#### 例子:
```java
/**
 * 一个锁和多个Condition(条件队列)可以更精细的控制并发
 */

package andy.com.concurrent.sync.cp;

import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class TestConsumerProducer_Lock {

    private static final Lock lock = new ReentrantLock();
    //用于放producer线程的条件队列
    private static final Condition producerCondition = lock.newCondition();
    //用于放consumer线程的条件队列
    private static final Condition consumerCondition = lock.newCondition();


    /**
     * @param args
     * @throws InterruptedException
     */
    public static void main(String[] args) throws InterruptedException {

        List<String> pool = new ArrayList<>();
        for (int i = 0; i < 2; i++) {
            new ConsumerL(pool, "consumer-" + i).start();
        }

        TimeUnit.SECONDS.sleep(5);

        for (int i = 0; i < 2; i++) {
            new ProducerL(pool, "producer-" + i).start();
        }

        TimeUnit.SECONDS.sleep(60 * 5);
    }


    /**
     * 生产者 往pool里面添加数据
     *
     * @author Admin
     */
    static class ConsumerL extends Thread {
        private List<String> pool = null;

        public ConsumerL(List<String> pool, String name) {
            this.pool = pool;
            this.setName(name);

            if (this.pool == null)
                throw new AssertionError("pool is null");
        }

        @Override
        public void run()  {
            while (true) {
                String i = null;
                lock.lock();
                try {
                    while (pool.size() == 0) {
                        System.out.println(this.getName() + " pool is empty waitting ");
                        //只在consumer 条件队列中等待
                        consumerCondition.await(); //
                    }

                    i = pool.remove(0);

                    //只唤醒Producer中的线程
                    producerCondition.signalAll();
                }
                catch (Exception e){ e.printStackTrace();}
                finally { lock.unlock(); }

                System.out.println(this.getName() + " consume " + i);

                //随机睡几秒
                sleepMs();
            }

        }

        private static void sleepMs() {
            try {
                TimeUnit.SECONDS.sleep(2 + new Random().nextInt(5));
            } catch (InterruptedException e) { e.printStackTrace(); }
        }
    }


    /**
     * 生产者 往pool里面添加数据
     *
     * @author Admin
     */
    static class ProducerL extends Thread {
        //max size of pool
        private static final int MAX_SIZE = 3;

        //shared pool
        private List<String> pool = null;

        public ProducerL(List<String> pool, String name) {
            this.pool = pool;
            this.setName(name);

            if (this.pool == null)
                throw new AssertionError("pool is null");
        }

        @Override
        public void run() {
            while (true) {
                lock.lock();
                try {
                    while (pool.size() == MAX_SIZE) {
                        System.out.println(this.getName() + " pool is full(" + pool.size() + ") waitting ");
                        //只在producer 条件队列中等待
                        producerCondition.await();
                    }

                    String i = new SimpleDateFormat("yyyyMMdd HH:mm:ss").format(new Date());
                    pool.add(i);

                    //只唤醒生产者线程
                    consumerCondition.signalAll();
                    System.out.println(this.getName() + " produce(" + pool.size() + ") " + i);
                }
                catch (Exception e){ e.printStackTrace();}
                finally { lock.unlock(); }
                sleepMs();
            }
        }


        private static void sleepMs() {
            try {
                TimeUnit.SECONDS.sleep(2 + new Random().nextInt(5));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }
}

```



### AQS: Abstract Queued Synchronizer
AQS 是一个框架：用来构建锁和Synchronizer的框架，Semaphore，ReentrantLock,FutureTask ,CountDownLatch等都是构建在 AQS之上的。  
AQS实现了一个Synchronizer的大量细节，比如等待线程的FIFO队列。
#### 两种操作:
AQS的所有操作都是不同形式的**获取(aquire)**和**释放(release)**，   
**获取操作**是**状态依赖的操作**，总能阻塞， **释放操作**不是一个可以阻塞的操作，但是可以运行在请求钱阻塞。  
获取操作可以是独占的，比如Rentrantlock；也可以是非独占的，比如Semaphore和CountdownLatch一样。
#### 一个状态
AQS有一个整数的状态信息state,状态信息可以通过protected的getState,setState,compareAndSetState来进行操作。比如 Semaphore用来state来表示剩余的许可数，FutureTask用state来表示人物状态(还未开始，运行，完成，取消)。

#### 我们看源代码 
##### 阻塞acquire的逻辑
不管是acquire还是acquireInterruptibly还是tryAcquireNanos都会调用 一个未被实现的 protected的 **tryAcquire**方法。
```java
  public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }


     /**
     * 最终都调用这个方法
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```


##### 非阻塞acquire的逻辑
不管是acquire还是acquireInterruptibly还是tryAcquireNanos都会调用 一个未被实现的 protected的 **tryAcquireShared**
```java

    /**
     * 最终都调用这个方法
     * 返回值:负数表示获取失败；0表示获取成功，但是后面线程获取将失败；正数表示获取成功，但是后面线程也将成功
     */
   protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }


    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

   
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }

   
```


##### 释放

```java

   
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    /**
     * Releases in shared mode.  Implemented by unblocking one or more
     * threads if {@link #tryReleaseShared} returns true.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryReleaseShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     * @return the value returned from {@link #tryReleaseShared}
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

     protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

     protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
   

```


#### AQS实现同步器只需实现四个方法
tryAcquire，tryAcquireShared,tryRelease,tryReleaseShared

#### 例子 OnShotLatch

只需要实现 tryAcquireShared 和 tryReleaseShared 十几行代码，就能实现一个简单的Synchronizer。

```java
package andy.com.concurrent.synchronizers.aqs;

import java.util.Date;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;

/**
 * 实现一个Latch，所有线程调用 sync.await 会阻塞，直到某个线程调用sync.signal打开闭锁
 */
class OnShotLatch {
    private final Sync sync = new Sync();
    
    public void await() throws InterruptedException {
        sync.acquireShared(0);
    }

    public void signal() {
        sync.releaseShared(0);
    }

    private class Sync extends AbstractQueuedSynchronizer {

        /**
         * 被 acquireShared 调用
         *
         * @param ignored
         * @return 小于0表示获取失败阻塞；0表示成功，后续获取操作会失败；大于0表示成功，后续获取操作也会成功
         */
        protected int tryAcquireShared(int ignored) {
            //-1活阻塞，1会成功
            return (getState() == 1) ? 1 : -1;
        }

        /**
         * 被releaseShared调用
         *
         * @param ignored
         * @return
         */
        protected boolean tryReleaseShared(int ignored) {
            setState(1); //闭锁打开
            return true; //现在其他线程可以取得闭锁
        }
    }

    public static void main(String[] args) {

        final OnShotLatch latch = new OnShotLatch();

        for (int i = 0; i < 5; i++) {

            Thread t = new Thread() {

                @Override
                public void run() {

                    try {
                        System.out.println(getName() + ":" + new Date().getTime() + ": wait for job");
                        latch.await();

                        System.out.println(getName() + ":" + new Date().getTime() + ": doing a job");
                        TimeUnit.SECONDS.sleep(3);

                        System.out.println(getName() + ":" + new Date().getTime() + ": finished a job");

                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            };
            t.setName("t-" + i);
            t.start();
        }

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println("打开闭锁开始工作");
        latch.signal();
    }
}

```


#### AQS 总结
AQS = acquire+release+state ，两类方法+一个状态
用AQS实现Synchronizer只需要实现tryAcquire，tryAcquireShared,tryRelease,tryReleaseShared四个方法并操作state变量。

#### AQS 具体实现
1. 同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

2. 同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型与名称以及描述如表55所示。

当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Nodeexpect,Nodeupdate)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联

 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/aqs-queue.png) 


 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/aqs-queue2.png) 


 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/aqs-queue3.png) 





 ```java

 
 /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
 1. 这个方法是每个使用aqs都要掉的方法，比如ReentrantLock 的lock.lock()方法就调用的 acquire(1).
 2. tryAcquire 这个方法需要用户实现，如果返回true表示获取锁成功，否则调用 **acquireQueued(addWaiter(Node.EXCLUSIVE), arg)** 经现场加入到队列尾部。
 下面是ReentrantLock的实现。
 如果c == 0 
 表示没有现场获取这个锁，直接使用 ompareAndSetState(0, acquires) 如果成功， 使用setExclusiveOwnerThread(current)设置当前线程互斥第获取锁。返回true。
如果 c ！= 0
表示锁已经被其他线程获取了， if (current == getExclusiveOwnerThread())表示是被自己获取的，直接返回true。

否则后去锁失败 返回 false。执行  **acquireQueued(addWaiter(Node.EXCLUSIVE), arg)**

```java
    //ReentrantLock 的非公平锁实现代码
     protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
 //ReentrantLock 的非公平锁实现代码
    final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

调用 acquireQueued(addWaiter(Node.EXCLUSIVE), arg) ，当前线程在“死循环”中尝试获取同步状态，而只有前驱节点是头节点才能够尝试获取同步状态。

```java
     /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor(); //前驱节点
                if (p == head && tryAcquire(arg)) { //注意这里也会调用tryAcquire方法
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```



想队列中加入node，compareAndSetTail(Nodeexpect,Nodeupdate)方法来确保节点能够被线程安全添加。在enq(finalNodenode)方法中，同步器通过“死循环”来保证节点的正确添加，在“死循环”中只有通过CAS将节点设置成为尾节点之后，当前线程才能从该方法返回，否则，当前线程不断地尝试设置。可以看出，enq(finalNodenode)方法将并发添加节点的请求通过CAS变得“串行化”了。


```sql

/**
     * Creates and enqueues node for current thread and given mode.
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }



    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```



释放锁的代码 释放锁，并唤醒双端队列中的下一个线程。
```java
    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);//唤醒后继者
            return true;
        }
        return false;
    }



     // ReentrantLock的tryRelease方法 
     protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null); 
            }
            setState(c);
            return free;
        }

```



### Condition
1. Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condition对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的，换句话说，Condition是依赖Lock对象的。

2. 
ConditionObject是同步器AbstractQueuedSynchronizer的内部类，因为Condition的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理。**每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。**

 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/condition1.png) 