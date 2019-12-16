---
layout: post
title: spring-cloud - hystrix
date: 2019-10-29 14:32:00
categories:  java spring springcloud hystrix
---
# Hystrix概览 
 **hystrix对应的中文名字是“豪猪”,豪猪周身长满了刺,能保护自己不受天敌的伤害,代表了一种防御机制,这与hystrix本身的功能不谋而合.**

## 1.Hystrix解决什么问题
1. 在复杂的分布式系统一个服务有可能会依赖有几十上百个外部服务，每个服务都不可避免的会出问题，如果这个服务不于依赖的外部服务做隔离处理，整个服务宕机的风险会非常高。
例如**服务A**依赖30个外部服务，如果30个外部服务有99.99%的时间是可用的。那么 99.99的30次方= 99.7%，整个系统的可用时间是99.7%。 那么10亿个请求就会有300万个请求失败。这是不可接受的。所以需要Hystrix来做。

2. 如果某个依赖服务的出了问题，比如预计20ms就会返回，但是耗时2s才返回，这就有可能会耗尽**服务A**的线程或者连接，影响其他接口或者服务。

## 2.hystrix做了什么
1. 防止单个依赖线程用尽容器线程(比如tomcat的工作线程)
2. 当出问题的时候快速失败，而不是将请求加入队列。 **直接返回不等了**
3. 当出问题的时候提供后备数据(兜底数据)，保护用户。**出问题了可以返回一个默认数据 而不是直接抛异常 这样就实现降级了**
4. 使用隔离技术(比如 bulkhead, swimlane, and circuit) 限制某一个依赖出问题的影响范围。
5. 通过近实时指标，监视和警报优化发现问题的时间 **可以设置没个时间窗的失败率来决定是否出问题**
6. 通过低延迟的配置修改，动态参数修改，优化恢复时间。  **可以设置参数，重试试探性的检查服务是否恢复**
7. 保护整个依赖客户端的执行，而不只是网络流量。 **可以设置整个command执行的时间(hystrix使用命令模式)。**

>1.Preventing any single dependency from using up all container (such as Tomcat) user threads.
>2.Shedding load and failing fast instead of queueing.
>3.Providing fallbacks wherever feasible to protect users from failure.
>4.Using isolation techniques (such as bulkhead, swimlane, and circuit breaker patterns) to limit the impact of any one dependency.
>5.Optimizing for time-to-discovery through near real-time metrics, monitoring, and alerting
>6.Optimizing for time-to-recovery by means of low latency propagation of configuration changes and support for dynamic property changes in most aspects of Hystrix, which allows you to make real-time operational modifications with low latency feedback loops.
>7.Protecting against failures in the entire dependency client execution, not just in the network traffic.

## 3. hystrix怎么做的
1. hystrix将原来调用封装到一个HystrixCommand或者HystrixObservableCommand中，在不同的线程中执行。
2. hystrix会为每个command执行设置时间，如果超过了上线，就timeout了。
3. 为每一个依赖调用维护一个小的线程池或者Semaphore，如果线程池满了或者Semaphore达到了上限制，就会直接拒绝执行
4. 监控成功，失败，超时
5. 使用熔断器，如果一个时间窗的错误率达到一定值，就跳闸
6. 当曹氏，执行失败，或者断路器跳闸了，提供后备数据。
7. 实时监控指标和配置

# 基本操作
## 原理
Hystrix使用**命令模式** 将逻辑全部封装到HystrixCommand中,在执行，可以有异步执行，同步执行两种。
## HelloWorld
```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import org.junit.Test;
import rx.Observable;
import rx.Observer;

import java.util.concurrent.Future;

public class CommandHelloWorld extends HystrixCommand<String> {

    private String name;

    public CommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
        return "hello " + name;
    }

    static public class TestCommandHelloWorld {

        /**
         * 同步调用
         */
        @Test
        public void testSyncHelloWord() {
            String name = "World";
            String s = new CommandHelloWorld(name).execute();
            assert s.equals("hello " + name);
        }

        /**
         * 异步同步调用
         * @throws Exception
         */
        @Test
        public void testAsyncHelloWord() throws Exception {
            String name = "World";
            Future<String> s = new CommandHelloWorld(name).queue();
            assert s.get().equals("hello " + name);
        }

        @Test
        public void testObservable() throws Exception {
            String name1 = "World";
            String name2 = "junjun";


            //阻塞调用
            Observable<String> ho = new CommandHelloWorld(name1).observe();
            Observable<String> co = new CommandHelloWorld(name2).observe();
            assert ho.toBlocking().single().equals("hello " + name1);
            assert co.toBlocking().single().equals("hello " + name2);


            //非阻塞调用
            ho.subscribe(new Observer<String>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {
                    e.printStackTrace();
                }

                @Override
                public void onNext(String s) {
                    System.out.println("onNext:" + s);
                }
            });


            co.subscribe(new Observer<String>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {
                    e.printStackTrace();
                }

                @Override
                public void onNext(String s) {
                    System.out.println("onNext:" + s);
                }
            });


            //java 8
            co.subscribe((s) -> {
                System.out.println("java 8 onNext:" + s);
            });
            co.subscribe((s) -> {
                        System.out.println("java 8 onNext:" + s);
                    },
                    (exception) -> {
                        exception.printStackTrace();
                    },
                    () -> {
                        System.out.println("java 8 conpleted");
                    });
        }
    }
}

```

## 2. Fallback 提供兜底数据实现降级
在执行 run的时候，除了抛出HystrixBadRequestException ，其他异常都会被看做是failure并触发getFallback方法
```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import org.junit.Test;

import java.util.concurrent.Future;

/**
 * 使用 fallback实现降级
 * 在执行 run的时候，除了抛出HystrixBadRequestException ，其他异常都会被看做是failure并触发getFallback方法，
 * 如果想要上层知道出了了异常，而不计入failure，可以将异常包装成HystrixBadRequestException再抛出。
 */
public class CommandHelloWorldWithFallback extends HystrixCommand<String> {

    private String name;

    public CommandHelloWorldWithFallback(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }
    /**
     * 抛出异常由实现了 getFallback 可以返回getFallback提供的数据。
     */
    @Override
    protected String run() throws Exception {
        throw new RuntimeException("failed ");
    }

    /**
     * 如果抛出异常会默认返回这个数据
     * @return
     */
    @Override
    public String  getFallback(){
        return "hello failure " + name;
    }


    static public class TestCommandHelloWorldFallback {

        @Test
        public void testSyncHelloWord() {
            String name = "World";
            String s = new CommandHelloWorldWithFallback(name).execute();
            System.out.println(s);
            assert s.equals("hello failure " + name);
        }

        @Test
        public void testAsyncHelloWord() throws Exception {
            String name = "World";
            Future<String> s = new CommandHelloWorldWithFallback(name).queue();
            System.out.println(s.get());
            assert s.get().equals("hello failure " + name);
        }
    }
}
```

## 3. 如果出了异常想要通知调用者
如果想要上层知道出了了异常，而不计入failure，可以在run中将异常包装成HystrixBadRequestException再抛出。

```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.exception.HystrixBadRequestException;
import org.junit.Test;

import java.util.concurrent.Future;

/**
 * 使用 fallback实现降级
 * 在执行 run的时候，除了抛出HystrixBadRequestException ，其他异常都会被看做是failure并触发getFallback方法，
 * 如果想要上层知道出了了异常，而不计入failure，可以将异常包装成HystrixBadRequestException再抛出。
 */
public class CommandHelloWorldHystrixBadRequestException extends HystrixCommand<String> {

    private String name;

    public CommandHelloWorldHystrixBadRequestException(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
        try {
            throw new RuntimeException("failed ");
        } catch (Exception e) {
            throw new HystrixBadRequestException("HystrixBadRequestException", e);
        }
    }

    @Override
    public String getFallback() {
        return "hello failure " + name;
    }


    static public class TestCommandHelloWorldHystrixBadRequestException {

        @Test
        public void testAsyncHelloWord() throws Exception {
            String name = "World";
            Future<String> s = new CommandHelloWorldHystrixBadRequestException(name).queue();
            System.out.println(s.get());
            assert s.get().equals("hello failure " + name);
        }
    }
}

```


## 4. Command Name、Command Group、Command Thead-Pool
### what
1. Command Name 默认为实现HystrixCommand的类名。
2. Command Group 是用来为Command分组的，默认一个组的Command都在同一个 Command Thread-Pool中执行。
3. Command 默认在设置的Command Group指定的Command Thead-Pool仲执行，也可以自己指定Thead-Pool

### 例子
同一个GroupName的Command指定不同的Thead-Pool
```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixThreadPoolKey;

/**
 * CommandName:一个command的名字
 * CommandGroup:用来组织Command，比如根据团队，库，报告等来自组织,名字是CommandGroupKey
 * Command Thread-Pool:如果不指定ThreadPoolKey，threadpool如果不指定，就是groupKey的名字。
 * 可以指定不同的ThreadPoolKey让command跑在不同的线程池
 */
public class CommandHelloWorldWithThreadPoolKey {

    static public class CommandHelloWorld1 extends HystrixCommand<String> {
        private String name;

        //同一个group但是在不同线程池
        private static Setter setter = Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("CommandHelloWorld1"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool1"));

        public CommandHelloWorld1(String name) {
            super(setter);
            this.name = name;
        }

        @Override
        protected String run() throws Exception {
            return "CommandHelloWorld1 hello " + name;
        }
    }

    static public class CommandHelloWorld2 extends HystrixCommand<String> {
        private String name;

        //同一个group但是在不同线程池
        private static Setter setter = Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("CommandHelloWorld2"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool2"));

        public CommandHelloWorld2(String name) {
            super(setter);
            this.name = name;
        }

        @Override
        protected String run() throws Exception {
            return "CommandHelloWorld2 hello " + name;
        }
    }

    public static void main(String args[]) {
        String name1 = "world";
        String name2 = "junjun";

        //下面两个在不同线程池中执行，起到隔离作用
        String s1 = new CommandHelloWorld1(name1).execute();
        String s2 = new CommandHelloWorld2(name2).execute();

        System.out.println(s1);
        System.out.println(s2);
    }
}

```

## 5.缓存
hystrix 支持缓存，缓存是存在一个 HystrixRequestContext中，HystrixRequestContext内部是一个ThreadLocal，
HystrixRequestContext 一般在web应用中，在request生命周期中使用，比如 在在filter中初始化，servlet调用后释放掉。
要支持缓存的话需要实现 getCacheKey()方法，这个方法返回一个缓存的key。

```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixThreadPoolKey;
import com.netflix.hystrix.strategy.concurrency.HystrixRequestContext;

/**
 * Hystrix 支持 cache, 需要实现 getCacheKey方法
 * cache 依赖于 HystrixRequestContext
 * Typically this context will be initialized and shut down via a ServletFilter that wraps a user request
 * or some other lifecycle hook.
 */
public class CommandHelloWorldWithCache extends HystrixCommand<Boolean> {

    private int value;

    public CommandHelloWorldWithCache(int value) {
        super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("CommandHelloWorld1"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("HelloWorldPool1")));
        this.value = value;
    }

    @Override
    protected Boolean run() throws Exception {
        return value == 0 || value % 2 == 0;
    }

    /**
     * cache key
     * @return
     */
    @Override
    public String getCacheKey(){
        return String.valueOf(value);
    }


    public static class Test {

        @org.junit.Test
        public void test1() {
            HystrixRequestContext ctx = HystrixRequestContext.initializeContext();
            try {

                CommandHelloWorldWithCache cmd1 = new CommandHelloWorldWithCache(2);
                CommandHelloWorldWithCache cmd2 = new CommandHelloWorldWithCache(2);

                assert cmd1.execute() == true;
                assert cmd1.isResponseFromCache() == false; //不是从cache中取的

                assert cmd2.execute() == true;
                assert cmd2.isResponseFromCache() == true; //是从cache中取的

            } finally {
                ctx.shutdown();
            }
        }
    }
}

```


## 5.请求合并 Request Collapsing
如果某个服务请求很频繁，可以将多个请求Command合并为一个，这样在一次command执行就可以完成，省去了多次调用的开销.特别是在网络调用时候很有效。
请求合并也是有时间限制的，默认两个10ms内的请求可以合并。
有两种方式的合并，1.request-scoped 2.globally-scoped，可以设置。

```java
package andy.com.springCloud.hystrix;

import com.netflix.hystrix.*;
import com.netflix.hystrix.strategy.concurrency.HystrixRequestContext;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.Future;
import java.util.stream.Collectors;

/**
 * request collpsing 请求折叠，将多个请求
 * HystrixCollapser<BatchReturnType, ResponseType, RequestArgumentType>
 */
public class CommandCollapserGetValueForKey extends HystrixCollapser<List<String>, String, Integer> {

    private Integer key;

    public CommandCollapserGetValueForKey(Integer key) {
        this.key = key;
    }

    @Override
    public Integer getRequestArgument() {
        return key;
    }

    @Override
    protected HystrixCommand<List<String>> createCommand(Collection<CollapsedRequest<String, Integer>> collection) {
        List<Integer> keys = collection.stream().map(t -> t.getArgument()).collect(Collectors.toList());
        return new BatchCommand(keys);
    }

    /**
     * 将批量执行获取的结果映射到单个请求
     *
     * @param strings
     * @param collection
     */
    @Override
    protected void mapResponseToRequests(List<String> strings, Collection<CollapsedRequest<String, Integer>> collection) {

        int i = 0;
        for (CollapsedRequest<String, Integer> req : collection) {
            req.setResponse(strings.get(i));
            i++;
        }
    }

    private static class BatchCommand extends HystrixCommand<List<String>> {

        private List<Integer> keys;


        public BatchCommand(List<Integer> keys) {
            super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ExampleGroup")));
            this.keys = keys;
        }

        @Override
        protected List<String> run() throws Exception {
            List<String> response = new ArrayList<>();

            for (Integer key : keys) {
                response.add("ValueForKey:" + key);
            }
            return response;
        }
    }
    
    public static class Test {

        @org.junit.Test
        public void test1() throws Exception {
            HystrixRequestContext ctx = HystrixRequestContext.initializeContext();
            try {

                Future<String> f1 = new CommandCollapserGetValueForKey(1).queue();
                Future<String> f2 = new CommandCollapserGetValueForKey(2).queue();
                Future<String> f3 = new CommandCollapserGetValueForKey(3).queue();

                //TimeUnit.SECONDS.sleep(5);
                Future<String> f4 = new CommandCollapserGetValueForKey(4).queue();

                String prefix = "ValueForKey:";
                assert f1.get().equals(prefix + 1);
                assert f2.get().equals(prefix + 2);
                assert f3.get().equals(prefix + 3);
                assert f4.get().equals(prefix + 4);

                Collection<HystrixInvokableInfo<?>> infos = HystrixRequestLog.getCurrentRequest().getAllExecutedCommands();
                for (HystrixInvokableInfo info : infos) {
                    System.out.println(info.getCommandKey() + ":" + info.getExecutionEvents());
                }
            } finally {
                ctx.shutdown();
            }
        }
    }
}

```

# 参考
1. [wiki](https://github.com/Netflix/Hystrix/wiki)