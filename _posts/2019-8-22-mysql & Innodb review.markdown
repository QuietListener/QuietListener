---
layout: post
title:  mysql和InnoDb Review
date:   2019-8-22 14:32:00
categories:  mysql innodb
---
# 1. mysql概览
## 1. Mysql的设计
mysql其实也是采用了分层设计 和 插件式设计。减小了耦合性
### 第1层:   
连接处理，认证授权，安全检查等
### 第2层:    
sql解析，分析，优化； 缓存；内置函数；存储过程，触发器，视图等。
### 第3层: （插件）  
存储引擎，存储引擎支付者数据的存储和提取，mysql定义了几十个底层API，存储引擎实现这些API(例如根据主键提取一行记录的api，开始一个事务的api)。这样存储引擎像插件一样工作。

## 2.mysql的并发控制
### 1. 采用"读写锁"来并发控制
#### 1. 读写锁的原理
1. 读锁是共享的，多个客户可以同时读取同一个资源。
2. 写锁是排他的，一个写锁会阻塞其他读锁和写锁，这样保证同一个时间只有一个用户能写入。   

在mysql中读锁也叫共享锁(**shared lock**) ，写锁叫排它锁(**exclusive lock**)

#### 2.锁的力度 
1. 表锁
   存储引擎可以实现自己的锁，但是mysql本身会使用表锁来实现不同的目的，例如alter table就会使用表锁。
2. 行锁
   行锁带来了更大的并发性，行锁在存储引擎实现，mysql不了解其中细节。

**InnoDB和XtraDB实现了行锁。**

### 3.事务
  **ACID**  Atomicity(原子性) Consistency(一致性) isolation(隔离线) Durability（持久性）

### 4.隔离级别(ACID中的Isolation)
隔离界别规定了**每一种隔离级别中“一个事务所做的修改，哪些是事务内、事务外可见的，哪些是不可见的。**
#### 1. READ UNCOMMITTED;
一个事务中所做的修改，即使还没有commit，对其他事务也是可见的。这叫脏读。

#### 2. READ COMMITTED;
 一个事务所做的修改(update)，只有commit，才对其他事务可见。但是对于”其他事务”，可能两次查询得到不一样的结果。 这叫“**不可重复读**”问题

#### 3. repeatable read 可重复读
Repeatable read 解决了“不可重复读”问题，在一个事务中，但是可能导致幻读问题。幻读问题是，当某个事务在读取某个范围内的记录时候，另一个事务又在该范围内插入了新行，之前的事务再次读取这个范围内的记录时候，会产生**幻行**。但是InnoDB和XtraDB使用MVCC解决了这个问题。

#### 4.serializable 串行化
解决所有问题，但是并发非常不友好


### 5. MVCC解决并发读写问题
#### 1.什么是MVCC
MVCC(multiple version concurrent control)多版本并发控制
mvcc通过保存数据在每某个时间点的快照来实现。在Innodb中的MVCC实现是通过**在每一行的后面加入两个隐藏列**来实现的。一个保存了行的**创建版本号**，一个保存了**过期版本**(删除时间)。每开始一个新事务，系统版本号都会自动递增。

#### 2. innodb中mvcc例子
repeatable-read 下mvcc怎么工作。注意下面几个版本号  

**当前事务版本号** ，**当前系统版本号**，**行的创建版本号** **行的删除版本号**
##### 1. select操作
a. 查找**创建版本号** 小于或等于 **当前事务版本号**。这样查找到的数据都是事务开始前就存在的。  
b. 查找 **删除版本号** 要么没有定义，要么大于**当前事务版本号**。这样找到的都是食物开始前没有被删除的。
##### 2. insert操作 
插入的新行的**创建版本号**设置为 **当前事务版本号** 
##### 3. delete操作  
为删除行的 **删除本号**设置为 **当前事务版本号**
##### 4. update操作   
插入一个新行，将**创建版本号**设置为**当前事务版本号**，设置原来行的**删除版本号**为 **当前事务版本号**

***innodb的MVCC实现大多数读都不用加锁，性能好。但是要牺牲存储空间,毕竟多了两个隐藏列。***




## 2. 索引
### 1. 基础
>   索引在数据量很大的时候 能加速 查询，提高几个数量级。
>   索引可以包含**1个列或者多个列**，
>   索引中列的**顺序**也很重要,msyql只能使用最左前缀列。
>   创建一个包含两个列的索引，和两个包含一个列的索引完全不一样。

### 2. 索引的基本数据结构(B-Tree)
   索引中对多个列进行排序的依据是和 **create table**中一致的,
```sql
CREATE TABLE `user_done_word_v2_t0` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `book_id` int(11) DEFAULT '0',
  `word_topic_id` int(11) NOT NULL,
  `first_at` int(11) DEFAULT '0',
  `score` int(11) DEFAULT '0',
  `wrong_times` int(11) DEFAULT '0',
  `done_times` int(11) DEFAULT '0',
  `total_used_time` int(11) DEFAULT '0',
  `name` varchar(255),
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `del` tinyint(1) DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_uid_bid_wid` (`user_id`,`book_id`,`word_topic_id`,),
  KEY `uid_wid_name` (`user_id`,`word_topic_id`,`name`)
  KEY `uid_wid_name_1` (`name`,`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8
```   
所以使用的时候 应该使用**左前缀**。
#### 可以使用索引的查询类型
##### 1.全职匹配
  例如:使用  这三个列 `user_id`,`word_topic_id`,`name`
##### 2.匹左配最   
  例如: 只使用 user_id 和 word_topic_id

##### 3. 匹配列的前缀
  例如 使用 KEY `uid_wid_name_1` (`name`,`user_id`) ，    
  查询name以”Jh“开头的人
##### 4. 匹配范围
   例如 使用 KEY `uid_wid_name_1` (`name`,`user_id`) ，    
   查询name在 ”Allen“和”Brown“之间的人
##### 5.精确匹配某左前列 并 范围匹配后一列。
   例如使用:   KEY `uid_wid_name` (`user_id`,`word_topic_id`,`name`)   
   查询 user_id = 1 word_topic_id=1   查询name在 ”Allen“和”Brown“之间的人

#### order by
由于索引是有序的，索引可以按某种方式查到值，也就一按这种方式用于排序。 满足上面1到5的查询类型，也满足排序需求。

#### 索引的限制(哪些情况不能使用索引)
##### 如果不是按照做前列开始查找，无法使用索引
 例如：select * from user_done_word_v2_t0 where word_topic_id = 1。不能使用索引
##### 不能跳过索引仲的列
   例如：select * from user_done_word_v2_t0 where user_id = 1 and name = "Jhonth"。就只能使用第一列。
##### 如果查询中有某个列有范围查询，它右边所有列都无法使用索引优化查询。




