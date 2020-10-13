---
layout: post
title: java并发(executor) review
date: 2019-10-29 14:32:00
categories:  java review
---

# 线程池  
## 1. Executor 
 ```java
 /**
 * An object that executes submitted {@link Runnable} tasks. This
 * interface provides a way of decoupling task submission from the
 * mechanics of how each task will be run, including details of thread
 * use, scheduling, etc.  An {@code Executor} is normally used
 * instead of explicitly creating threads. For example, rather than
 * invoking {@code new Thread(new(RunnableTask())).start()} for each
 * of a set of tasks, you might use:
 ......
 */
 public interface Executor {
    void execute(Runnable command);
}
```

## 2. executor 的初衷
 
 >An object that executes submitted {@link Runnable} tasks. This
 > interface provides a way of decoupling task submission from the
 > mechanics of how each task will be run, including details of thread
 > use, scheduling, etc.  An {@code Executor} is normally used
 > instead of explicitly creating threads. 

 Executors是用来提交任务，这个接口将任务提交和任务运行机制解耦，这些机制可以是运行任务的线程，怎么调度等。 用来替代显示的创建线程。


## 3. ExecutorService
```java

/**
 * An {@link Executor} that provides methods to manage termination and
 * methods that can produce a {@link Future} for tracking progress of
 * one or more asynchronous tasks.

...
...
 */
public interface ExecutorService extends Executor {

    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    
    /**
    * 省略更多的函数
    */
}
```
###  ExecutorService 的初衷
 >An {@link Executor} that provides methods to manage termination and
 > methods that can produce a {@link Future} for tracking progress of
 > one or more asynchronous tasks.   

ExecutorService是一个Executor（继承关系），它提供了管理关闭和跟踪一个或者多个异步任务执行结果(Future)的方法。 是一个加强版的Executor,提供了生命周期管理功能。


### 3. Executors
```java

/**
 * Factory and utility methods for {@link Executor}, {@link
 * ExecutorService}, {@link ScheduledExecutorService}, {@link
 * ThreadFactory}, and {@link Callable} classes defined in this
 * package. This class supports the following kinds of methods:
 *
 * <ul>
 *   <li> Methods that create and return an {@link ExecutorService}
 *        set up with commonly useful configuration settings.
 *   <li> Methods that create and return a {@link ScheduledExecutorService}
 *        set up with commonly useful configuration settings.
 *   <li> Methods that create and return a "wrapped" ExecutorService, that
 *        disables reconfiguration by making implementation-specific methods
 *        inaccessible.
 *   <li> Methods that create and return a {@link ThreadFactory}
 *        that sets newly created threads to a known state.
 *   <li> Methods that create and return a {@link Callable}
 *        out of other closure-like forms, so they can be used
 *        in execution methods requiring {@code Callable}.
 * </ul>
 *
 * @since 1.5
 * @author Doug Lea
 */
public class Executors {
}

```
 Executors 是一个工具类(工厂类); TA为Executor,ExecutorService ,ScheduledExecutorService,等提供支持。


### 3. ThreadPoolExecutor 线程池的具体实现。
ThreadPoolExecutor 是线程池的具体实现，他使用一个BlockQueue作为任务队列，
executor只需要从队列里面获取任务然后执行即可。
参考:[java线程池的注意事项](http://quietlistener.github.io/java/%E7%BA%BF%E7%A8%8B%E6%B1%A0/2019/06/01/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9.html)


### 4.几种默认类型的线程池(一遍都不用)。
#### 1.newFixedThreadPool 定长线程池+无限队列
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
没添加一个任务，新建一个线程，直到达到最大线程数，并一直保持这个线程数。 任务队列无限大。   
**这种线程池通过将任务缓存来应对突发的任务提交。TA的伸缩性体现在任务队列**

#### 2. newCachedThreadPool 无限线程池+有限队列
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
```
1. SynchronousQueue 可以看做是长度为1的Queue，只能容纳一个元素。但是他的效率比ArrayBlockQueue要高。并且提供公平机制(FIFO).可以参考[demo代码](https://github.com/QuietListener/andy.com/blob/master/src/main/java/andy/com/concurrent/queue/SynchronousQueueTest.java)
2. maxPoolSize为Integer.MAX_VALUE可以看做是无限大。所有当有一个任务提交，系统如果没有空闲进程，会新建一个进程来处理任务。
3. keepAliveTime 为60秒，这种线程池可以在后面回收不用的线程。
**这种线程池通过加线程应对突发的任务提交。TA的伸缩性体现在增加线程和回收线程**


### 残酷的现实：不推荐使用Executors创建线程
使用newFixedThreadPool，newCachedTheadPool 都有可能造成OOM，newFixedThreadPool的队列是无限长的， newCachedTheadPool可以创建无限个(Integer.MAX_VALUE)线程。**(阿里巴巴Java开发手册中中也是禁止的)**
正确的姿势是使用 ThreadPoolExecutor自己制定 线程的个数与BlockQueue。 BlockQueue有两种，ArrayBlockQueue和LinkedBlockQueue。


### 执行周期性的任务 ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor 继承自 ThreadPoolExecutor ，可以执行定期任务和周期性任务。
可以参考[demo代码](https://github.com/QuietListener/andy.com/blob/master/src/main/java/andy/com/concurrent/executor/TestScheduledThreadPoolExecutorService.java) 他使用的队列是延迟队列(DelayQueue) 。   

```java
public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }
```
延迟队列可以参考[DelayQueue demo](https://github.com/QuietListener/andy.com/blob/master/src/main/java/andy/com/concurrent/containers/TestDelayQueue.java)




# 线程的结束
java线程的stop已经Deprecated了，推荐使用interrupt来结束线程,这是一种更优雅和灵活的方式，thread1.interrupt()的调用只是通知目标线程需要停止，至于什么时候停止，是目标线程自己决定在什么时机停止。

线程有一个interuptStatus 它的初始状态为false。他的值可能有下面几种情况
```java
/**
     * Interrupts this thread.
     *
     * <p> Unless the current thread is interrupting itself, which is
     * always permitted, the {@link #checkAccess() checkAccess} method
     * of this thread is invoked, which may cause a {@link
     * SecurityException} to be thrown.
     *
     * <p> If this thread is blocked in an invocation of the {@link
     * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
     * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
     * class, or of the {@link #join()}, {@link #join(long)}, {@link
     * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
     * methods of this class, then its interrupt status will be cleared and it
     * will receive an {@link InterruptedException}.
     *
     * <p> If this thread is blocked in an I/O operation upon an {@link
     * java.nio.channels.InterruptibleChannel InterruptibleChannel}
     * then the channel will be closed, the thread's interrupt
     * status will be set, and the thread will receive a {@link
     * java.nio.channels.ClosedByInterruptException}.
     *
     * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
     * then the thread's interrupt status will be set and it will return
     * immediately from the selection operation, possibly with a non-zero
     * value, just as if the selector's {@link
     * java.nio.channels.Selector#wakeup wakeup} method were invoked.
     *
     * <p> If none of the previous conditions hold then this thread's interrupt
     * status will be set. </p>
     *
     * <p> Interrupting a thread that is not alive need not have any effect.
     *
     * @throws  SecurityException
     *          if the current thread cannot modify this thread
     *
     * @revised 6.0
     * @spec JSR-51
     */
    public void interrupt() {
        ...
    }
```

1. 当处于wait join sleep 的阻塞状态时候，线程会将interruptStatus 复位(设置为false)，然后抛出 InterruptedException。
2. 如果是阻塞在I/O,interruptStatus 会被设置为true,并抛出ClosedByInterruptException异常，斌立即返回。
3. 如果阻塞在java.nio.channels.Selector，interruptStatus 会被设置为true。


 