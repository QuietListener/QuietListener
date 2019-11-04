---
layout: post
title: java并发(executor) review
date: 2019-10-29 14:32:00
categories:  java review
---

# 1. Executor 框架
 ### 1. Executor 
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
 #### 1. executor 的初衷
 >An object that executes submitted {@link Runnable} tasks. This
 > interface provides a way of decoupling task submission from the
 > mechanics of how each task will be run, including details of thread
 > use, scheduling, etc.  An {@code Executor} is normally used
 > instead of explicitly creating threads. 

 Executors是用来提交任务，这个接口将任务提交和任务运行机制解耦，这些机制可以是运行任务的线程，怎么调度等。 用来替代显示的创建线程。


### 2. ExecutorService
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
#####1. ExecutorService 的初衷
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
 Executors 是一个工具类(工厂类);TA为Executor,ExecutorService ,ScheduledExecutorService,等提供支持。


### 3. ThreadPoolExecutor 线程池的具体实现。
参考:[java线程池的注意事项](http://quietlistener.github.io/java/%E7%BA%BF%E7%A8%8B%E6%B1%A0/2019/06/01/java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9.html)


 线程池的参数
 corePoolSize vs maxPoolSize
 线程池的关闭:

 interupt:理解
 

# 参考
1. [SpringCloud微服务如何优雅停机及源码分析](https://www.cnblogs.com/trust-freedom/p/10744683.html)
1. [SpringCloud服务的平滑上下线](https://juejin.im/post/5cf63899f265da1b9253c7f4)
