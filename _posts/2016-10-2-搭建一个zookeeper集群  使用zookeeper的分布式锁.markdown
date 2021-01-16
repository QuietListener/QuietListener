---
layout: post
title:  搭建一个zookeeper集群 使用zookeeper的分布式锁
date:   2016-10-2 14:32:00
categories: zookeeper 分布式
---

### 1. 下载 安装
 ```
 wget http://mirrors.hust.edu.cn/apache/zookeeper/stable/zookeeper-3.4.9.tar.gz
 tar -zxvf zookeeper-3.4.9.tar.gz 
 cd zookeeper-3.4.9
 cd conf/
 cp zoo_sample.cfg  zoo.cfg
```

 **使用下面这个目录存放zookeeper的内存映像和log**

 mkdir /var/www/data/zookeeper
 
 ### 2. 配置文件
 vim zoo.cfg 
 #### 单机版的
 ```

# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/var/www/data/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```

#### 集群版的
```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/var/www/data/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

server.1=cinder:2888:3888
server.2=rc2:2888:3888
server.3=talentbigdata:2888:3888
```

**集群的话还要在dataDir=/var/www/data/zookeeper 下面新建一个myid文件，内容是一个数字。比如cinder上的话，就是1（server.1=cinder:2888:3888）**


启动zk服务器
```
[www@iZ23kzithsiZ zookeeper-3.4.9]$ bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /home/www/programs/zookeeper-3.4.9/bin/../conf/zoo.cfg


Starting zookeeper ... STARTED
```

**启动客户端**
```
[www@iZ23kzithsiZ zookeeper-3.4.9]$ bin/zkCli.sh 
Connecting to localhost:2181
2016-11-15 16:57:33,485 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2016-11-15 16:57:33,489 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=iZ23kzithsiZ
2016-11-15 16:57:33,490 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_73
2016-11-15 16:57:33,492 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2016-11-15 16:57:33,492 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/home/www/programs/jdk1.8.0_73/jre
2016-11-15 16:57:33,492 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/home/www/programs/zookeeper-3.4.9/bin/../build/classes:/home/www/programs/zookeeper-3.4.9/bin/../build/lib/*.jar:/home/www/programs/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/home/www/programs/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/home/www/programs/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/home/www/programs/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/home/www/programs/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/home/www/programs/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/home/www/programs/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/home/www/programs/zookeeper-3.4.9/bin/../conf:
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=2.6.32-431.23.3.el6.x86_64
2016-11-15 16:57:33,493 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=www
2016-11-15 16:57:33,494 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/www
2016-11-15 16:57:33,494 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/home/www/programs/zookeeper-3.4.9
2016-11-15 16:57:33,495 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@67424e82
Welcome to ZooKeeper!
JLine support is enabled
[zk: localhost:2181(CONNECTING) 0] 2016-11-15 16:57:35,562 [myid:] - INFO  [main-SendThread(64.127.0.0:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server 64.127.0.0/64.127.0.0:2181. Will not attempt to authenticate using SASL (unknown error)
```

Zookeeper的命令
```
[zk: localhost:2181(CONNECTED) 0] help
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port
```


**检查各个节点状态**
```
[www@iZ23kzithsiZ zookeeper-3.4.9]$ ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/www/programs/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: follower
[www@iZ23kzithsiZ zookeeper-3.4.9]$ 
```


### 2. 封装了一个ruby的分布式锁
```
#encoding:utf-8

#zookeeper 封装的lock
class ZkLock
    def initialize(host,port,path=nil,mode = :ephemeral ) #mode默认为“临时的”
      raise Exception.new("host = #{host}; port = #{port}; path = #{path}") if host.blank? or path.blank?
      @zk = ZK.new("#{host}:#{port}")
      @queue = Queue.new
      @path = path
      @mode = mode
      @info = "lock";
    end

    def with_lock
      begin
        @zk.create(@path, @info,:mode => @mode)
        yield if block_given?
      ensure
        @zk.close!
      end
    end
end
```
使用锁
```
zk = ZkLock.new(zk_host, zk_port, path)
zk.with_lock do
##业务逻辑
end

```

看看这篇
https://juejin.cn/post/6844903729406148622

