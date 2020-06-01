---
layout: post
title: thrift的io模型和线程模型以及组织架构
date: 2020-5-8 14:32:00
categories:  java maven
---

# 1. why thrift
公司最早开始使用thrift是在2014年，在高速发在期间， 迭代非常快。 当时为了减少沟通成本，加快迭代速度。考虑了技术成熟度(facebook出品)，决定使用thrift开发。

# 2.io模型和线程模型
公司使用的是TThreadedSelectorServer，在单机高并发方面还是非常给力。
这里总结一下thrift的io模型和线程模型，这里用io来划分，分为NIO和BIO两部分

## 1. BIO（阻塞io）
### 1. TSimpleServer 
读写使用阻塞io，accept/read/write和业务逻辑都在一个线程里完成。多用来测试

### 2. TThreadPoolServer 
读写使用阻塞io，accept在一个单独线程，每建立一个链接，将所有的read/write 和业务逻辑放到一个线程池处理。

## 2. NIO
### 1. TNonblockingServer 
使用非阻塞io, 单个selector， select/accept/read/write和业务逻辑都在一个线程中处理。

### 2. THsHaServer 
使用非阻塞io, 单个selector， select/accept/read/write在一个线程，业务逻辑放入到一个线程池处理。
HsHa = Half Sync/Half Async ，我自己理解为read，write，accept这些是同步，业务处理是异步，所以是半同步半异步。

### 3. TThreadedSelectorServer 
使用非阻塞io, 一个selector在AcceptThread线程中处理accept事件，然后多个SelectorThread线程(每个线程一个selector,名字叫)处理read/write事件，业务逻辑放入到一个线程池处理(WorkerThread)。
这样更好的利用了多核cpu的能力，获取更好的并发性,也是线上使用的Server。


# 3. 这几个server的继承关系如下图
![thrift](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-06-01-thrift-io-thread.jpg)

	

# 2.架构
看thrift的源码，很明显thrift是一个分层架构，层与层之间定了接口，某一层可以有不同的实现，是一个不错的软件架构。

### 1.最底层是Transport层
这一层是io层，负责数据的读写，这一层要实现的代码是
read write flush 等io功能。
```java
/**
 * Generic class that encapsulates the I/O layer. This is basically a thin
 * wrapper around the combined functionality of Java input/output streams.
 *
 */
public abstract class TTransport implements Closeable {
 public abstract int read(byte[] buf, int off, int len)
    throws TTransportException;
}
public void write(byte[] buf) throws TTransportException {
    write(buf, 0, buf.length);
  }
public void flush()
    throws TTransportException {}  
}
```
   
### 2. 再上一层是协议层 TProtocol
1. 协议层从transport层读取byte，并负责解释这些数据，将数据封装为TMessage，TStruct,TMap，String等结构哈数据    
2. 并且负责将结构化数据写入到指定的transport。
3. 从源码看 TProtocol构造函数需要TTransport。

```java

/**
 * Protocol interface definition.
 *
 */
public abstract class TProtocol {

  protected TProtocol(TTransport trans) {
    trans_ = trans;
  }

  /**
   * Transport accessor
   */
  public TTransport getTransport() {
    return trans_;
  }

  /**
   * Writing methods.
   */

  public abstract void writeMessageBegin(TMessage message) throws TException;

  public abstract void writeMessageEnd() throws TException;

  public abstract void writeStructBegin(TStruct struct) throws TException;

  public abstract void writeStructEnd() throws TException;
 ....

  /**
   * Reading methods.
   */

  public abstract TMessage readMessageBegin() throws TException;

  public abstract void readMessageEnd() throws TException;

  public abstract TStruct readStructBegin() throws TException;

  public abstract void readStructEnd() throws TException;

  public abstract TField readFieldBegin() throws TException;

  public abstract void readFieldEnd() throws TException;
}
```  

### 3. 业务处理层TProcessor
TProcessor处理具体的业务逻辑，他需要依赖TProtocal

```java
/**
 * A processor is a generic object which operates upon an input stream and
 * writes to some output stream.
 *
 */
public interface TProcessor {
  public boolean process(TProtocol in, TProtocol out)
    throws TException;
}
```

**下面是TBaseProcessor的一个一个具体例子**
```java

public abstract class TBaseProcessor<I> implements TProcessor {
  private final I iface;
  private final Map<String,ProcessFunction<I, ? extends TBase>> processMap;

  protected TBaseProcessor(I iface, Map<String, ProcessFunction<I, ? extends TBase>> processFunctionMap) {
    this.iface = iface;
    this.processMap = processFunctionMap;
  }

  public Map<String,ProcessFunction<I, ? extends TBase>> getProcessMapView() {
    return Collections.unmodifiableMap(processMap);
  }

  @Override
  public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin(); //读取方法
    /*
    * processMap是具体业务逻辑名字作为key，具体方法作为value的一个Map
    * processMap.get(msg.name);获取的具体方法
    */
    ProcessFunction fn = processMap.get(msg.name);

    if (fn == null) {//fn==null 找不到方法，报异常
      TProtocolUtil.skip(in, TType.STRUCT);
      in.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
      x.write(out);
      out.writeMessageEnd();
      out.getTransport().flush();
      return true;
    }

    fn.process(msg.seqid, in, out, iface); //处理具体业务逻辑
    return true;
  }
}

```