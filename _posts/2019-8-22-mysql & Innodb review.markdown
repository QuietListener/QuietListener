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
 一个事务所做的修改(update)，只有commit，才对其他事务可见。但是对于”其他事务”，可能两次查询同一条记录得到不一样的结果。 这叫“**不可重复读**”问题

#### 3. repeatable read 可重复读
Repeatable read 解决了“不可重复读”问题，但是可能导致幻读问题。幻读问题是，当某个事务在读取**某个范围内**的记录时候，另一个事务又在该范围内插入了新行，之前的事务再次读取这个范围内的记录时候，得到不一样的行数，会产生**幻行**。但是InnoDB和XtraDB使用MVCC解决了这个问题。

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

注意**快照读**和**当前读** https://juejin.im/post/5c9040e95188252d92095a9e


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
### 可以使用索引的查询类型
#### 1.全职匹配
  例如:使用  这三个列 `user_id`,`word_topic_id`,`name`
#### 2.匹左配最   
  例如: 只使用 user_id 和 word_topic_id

#### 3. 匹配列的前缀
  例如 使用 KEY `uid_wid_name_1` (`name`,`user_id`) ，    
  查询name以”Jh“开头的人
#### 4. 匹配范围
   例如 使用 KEY `uid_wid_name_1` (`name`,`user_id`) ，    
   查询name在 ”Allen“和”Brown“之间的人
#### 5.精确匹配某左前列 并 范围匹配后一列。
   例如使用:   KEY `uid_wid_name` (`user_id`,`word_topic_id`,`name`)   
   查询 user_id = 1 word_topic_id=1   查询name在 ”Allen“和”Brown“之间的人

### order by
由于索引是有序的，索引可以按某种方式查到值，也就一按这种方式用于排序。 满足上面1到5的查询类型，也满足排序需求。

### 索引的限制(哪些情况不能使用索引)
#### 如果不是按照做前列开始查找，无法使用索引
 例如：select * from user_done_word_v2_t0 where word_topic_id = 1。不能使用索引
#### 不能跳过索引仲的列
   例如：select * from user_done_word_v2_t0 where user_id = 1 and name = "Jhonth"。就只能使用第一列。
#### 如果查询中有某个列有范围查询，它右边所有列都无法使用索引优化查询。


### 索引的一些最佳实践与坑
#### 1.在函数中使用索引列，msyql不会使用索引加速查询
```sql
CREATE TABLE `People` (
  `last_name` varchar(50) NOT NULL,
  `first_name` varchar(55) NOT NULL,
  `dob` date NOT NULL,
  `gender` enum('m','f','u') NOT NULL DEFAULT 'u',
  KEY `last_name` (`last_name`,`first_name`,`dob`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

直接使用索引列
```sql
mysql> explain select * from People where last_name = "a";
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | People | NULL       | ref  | last_name     | last_name | 52      | const |    2 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```


在函数中使用索引列
```sql
mysql> explain select * from People where CHARACTER_LENGTH(last_name) = 1;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | People | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```


#### 2. 长字符串的前缀索引
text，varchar，bolb 字符串很长，有时候只需要前面N个字符也能有很好的区分度，不需要全部索引。
可以使用 **KEY `first_name` (`first_name`(3))** 添加索引  

或者使用 **alter table People add key (first_name(3))**

```sql
CREATE TABLE `People` (
  `last_name` varchar(50) NOT NULL,
  `first_name` varchar(55) NOT NULL,
  `dob` date NOT NULL,
  `gender` enum('m','f','u') NOT NULL DEFAULT 'u',
  KEY `last_name` (`last_name`,`first_name`,`dob`),
  KEY `first_name` (`first_name`(3))
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```


添加前缀索引前  ，没有使用索引，type也是All(全表扫描)  
```sql
mysql> explain select * from  People where first_name like "ddd%"
    -> ;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | People | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   10 |    11.11 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

添加前缀索引后  ,我们看到使用了索引first_name,type也变成了range。  
```sql
mysql> alter table People add key (first_name(3));
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select * from  People where first_name like "ddd%"
    -> ;
+----+-------------+--------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | People | NULL       | range | first_name    | first_name | 5       | NULL |    1 |   100.00 | Using where |
+----+-------------+--------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

#### 3.每列都是索引，不太好
```sql
CREATE TABLE `People1` (
  `last_name` varchar(50) NOT NULL,
  `first_name` varchar(55) NOT NULL,
  `dob` date NOT NULL,
  `gender` enum('m','f','u') NOT NULL DEFAULT 'u',
  KEY `last_name` (`last_name`),
  KEY `first_name` (`first_name`),
  KEY `dob` (`dob`),
  KEY `gender` (`gender`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

使用or 连接的两个索引，其实没有使用索引，任然是全表扫描(type是ALL).
```sql
mysql> explain select * from People1 where last_name = "a" or first_name="b";
+----+-------------+---------+------------+------+----------------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys        | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+----------------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | People1 | NULL       | ALL  | last_name,first_name | NULL | NULL    | NULL |   10 |    40.00 | Using where |
+----+-------------+---------+------------+------+----------------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

使用 and 也只能用到一个索引:
```sql
mysql> explain select * from People1 where last_name = "a" and first_name="b";
+----+-------------+---------+------------+------+----------------------+------------+---------+-------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys        | key        | key_len | ref   | rows | filtered | Extra       |
+----+-------------+---------+------------+------+----------------------+------------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | People1 | NULL       | ref  | last_name,first_name | first_name | 57      | const |    1 |    40.00 | Using where |
+----+-------------+---------+------------+------+----------------------+------------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

#### 4.选择索引列的顺序
  **原则上将区分度更大的列放在最前面。**
  比如有两个列 c1 和 c2; 
  ```sql
  select count(disctinct(c1))/count(*) as c1_select,  count(disctinct(c2))/count(*) as c2_select from xxx ;
  ```
  返回:   
  ```sql
  c1_select : 0.001  
  c2_select : 0.2
  ```
就应该选择c2为第一列。
**注意事项**
选了区分度更大的列也可能有问题，比如c2。  
假设c2有一个值占了10%的量，比如c2是用户类型，c2有一个值是试用用户，有20%的用户是试用用用户，假设一个表有2000w行，使用c2查询使用用户时候会扫描200w行数据，使用了c2索引，也会非常慢。这就需要程序上做特殊处理了。


#### 5.聚族索引
  聚族索引是一种数据存储方式。每个表只能有一个聚族索引(就是Primary Key).InnoDb中在叶子节点保存所有的数据，非叶子节点只保存索引列。
##### 1. 聚族索引的好处:
1. 包相关数据放在一起(物理上一起)，按照聚族索引来查找的话，比如按用户Id这个主键来查找的话，由于相邻用户Id的用户数据放在一起，只需要读取很少的磁盘块就能拿到用户数据。减少了IO。
2. 按照主键顺序插入非常快。

##### 2. 聚族索引的缺点。
1. 如果数据小，全部在内存中，没有太大优势。
2. 主键更新或者插入新行后，面临”主键分裂“问题。当主键要求必须将某一行插入到一个已满的”页“中的时候，需要将这个页分裂为两个页，页分裂会占用更多的空间。
3. 非聚族索引会更大，因为耳机索引叶子节点不是存的具体数据，而是引用行的主键。
4. 二级索引要慢一些，二级索引找到数据的主键后，再使用主键找到具体的数据的具体位置。



#### 6. 覆盖索引
我们在查找的数据直接来自索引，不用回表来获取数据。  

下面是mysql工作的过程。
>mysql使用索引扫描是很快的，只需要从一条记录移动到下一条记录，如果索引不能覆盖所有的查询所需的全部列，就不得不烧苗一条索引就回表查询一次对应的行。  

```sql
CREATE TABLE `People1` (
  `last_name` varchar(50) NOT NULL,
  `first_name` varchar(55) NOT NULL,
  `dob` date NOT NULL,
  `gender` enum('m','f','u') NOT NULL DEFAULT 'u',
  KEY `last_name` (`last_name`),
  KEY `first_name` (`first_name`),
  KEY `dob` (`dob`),
  KEY `gender` (`gender`),
  KEY `aaa` (`last_name`,`first_name`,`dob`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```
```sql
mysql> explain select dob  from People1 where dob < "2019-08-04" ;
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
| id | select_type | table   | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | People1 | NULL       | range | dob           | dob  | 3       | NULL |    8 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```
比如我们有一个key是dob，我们也只需要dob。Extra中就有"Using index"表示数据直接来自索引。   



#### 7.使用索引烧苗来排序
1. mysql可以使用同一个索引既满足排序，又满足查找行，应该尽可能设计得同时满足这两个任务。  
2. 当索引的列顺序和order by字句孙旭完全一致，并且所有列的排序方向都一样时，mysql可以使用索引进行排序。
3. 如果关联了多张表，只有order by 字句的引用全部为第一个表示，才能使用索引做排序。
4. order by 和查询一样 需要满足左前缀
5. 如果在条件中使用了常量，可以使用第二列进行排序

```sql
CREATE TABLE `People1` (
  `last_name` varchar(50) NOT NULL,
  `first_name` varchar(55) NOT NULL,
  `dob` date NOT NULL,
  `gender` enum('m','f','u') NOT NULL DEFAULT 'u',
  KEY `last_name` (`last_name`),
  KEY `first_name` (`first_name`),
  KEY `dob` (`dob`),
  KEY `gender` (`gender`),
  KEY `aaa` (`last_name`,`first_name`,`dob`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

last_name使用常量，order by可以使用第二列 first_name来排序。
```sql
select * from People1 where last_name="johin" order by first_name;
```


#### 8. 索引和锁
InnoDB只有在访问行的时候才会对行加锁，所以多使用索引，就减少InnoDB访问的行数，从而减少锁的数量。


#### 9. 最佳实践。
1. 尽量将范围查询的列放在 索引 **右边**。因为mysql不会使用范围查询以后的索引列  
2. 像性别这种 “选择性不高” 的列也可以放在前面，因为我们可以使用 **where sex in('m', 'f') and ....** 来规避。这样虽然sex选择性不高，但是把ta放在索引第1的位置，也不影响我们使用索引后面的列来加快速度。
3. 避免多个范围条件，因为对于索引来说，只能使用一个范围条件。

4. 排序,分页
如果有 索引(sex,rating)下面的查询依然很慢。因为mysql会哈很多时间来扫描丢弃的行。  
```sql
 select * from XXX where sex="1" order by rating desc limit 10000，10.
```

上面的查询很慢 limit后的数很大，需要扫描很多的行。  我们可以使用 ***延迟关联*** 的技术。先找到 ( 10000，10010)的主键，再进行关联。
```sql
select * from XXX inner join (select id from XXX where sex="1" order by rating desc limit 10000，10) as x using(id)
```
 5. optimize table 消除碎片







# Innodb概述:
## 1. innodb特点:
1. 完整支持ACID事务
2. 实现了四种隔离级别
3. 行锁设计，通过mvcc来获得高并发(一致性的非锁定读，读的时候不用加锁)。
4. 在隔离级别为Repeatable Read，使用next-key locking的策略来避免幻读。
5. 存储引擎使用 插入缓冲(inser buffer),二次写(double write),自适应哈希索引(adaptive hash index),预读(read ahead)等高性能高可用功能。
6. 使用聚集(clustered)的方式，来存放每张表的数据，所以每张表的数据都是按主键顺序存放，没有主键的表也会默认为生成一个6字节的rowId作为主键。




## 2.mysql一些信息

### 1. 配置文件
mysql启动时候会去/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf  这些地方找配置文件，如果有多个文件存在以最后一个读到的文件参数为准
```shell
$ mysql --help | grep my.cnf   
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf 
```
### 2. datadir指定数据库所在路径
```shell
mysql> show variables like 'datadir';
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| datadir       | /usr/local/mysql/data/ |
+---------------+------------------------+
1 row in set (0.02 sec)

```



### 3.  mysql的体系结构:
1. mysql的线程模型是**单进程多线程的**
2. mysql是分层的
3. mysql的存储引擎采用的是插件式的。

![mysql-architect](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-06-03-mysql-archtec.jpg)


**在 repeatable-read innodb使用 next-key locking的策略来避免幻读。**


mysql 支持的存储引擎
```shell
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+

```

### 4. 使用套接字链接mysql
```shell
mysql> show variables like 'socket';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| socket        | /tmp/mysql.sock |
+---------------+-----------------+
1 row in set (0.14 sec)

mysql> 

```

可以直接使用套接字链接mysql
```shell
➜  Documents mysql -uroot -p -S /tmp/mysql.sock  
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 352
Server version: 5.7.17 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```


## 3 . InnoDb的架构
innodb是采用单进程多线程模式 ，innodb有很多内存块，这些内存块组成了和一个内存池。InnoDb负责以下工作:
1.维护进程/线程需要访问的多个内部数据结构
2.缓存磁盘上的数据，方便快速读取，并对在磁盘文件的数据进行修改之前，先在这里修改。
3.重做日志(redo log)的缓冲

可以把InnoDb分为2部分，一部分是后台线程一部分是内存池
架构如下:
![innodb-architect](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-06-03-innodb-architec.jpg)

### 1. 后台线程

#### 1. **master thread**:
 最核心的线程，主要负责缓冲词的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新，合并插入缓冲(insert buffer),undo也的回收等。
####  2. **io线程:** 
  使用aio来处理写IO请求，IO线程主要处理这些io请求的回调。
 Io线程包括：insert buffer , log  ,read  和 write 线程,可以使用innodb_read_io_theads 和 innodb_write_io_theads设置read和write线程的数量。
 

使用下面命令可以看到各个线程的情况：
可以看到有4个read和4个write线程。
```
mysql> show engine innodb status \G;
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2020-06-03 11:06:54 0x70000ce64000 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 3 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 588 srv_active, 0 srv_shutdown, 909653 srv_idle
srv_master_thread log flush and writes: 902000
----------
....
--------
FILE I/O
--------
I/O thread 0 state: waiting for i/o request (insert buffer thread)
I/O thread 1 state: waiting for i/o request (log thread)
I/O thread 2 state: waiting for i/o request (read thread)
I/O thread 3 state: waiting for i/o request (read thread)
I/O thread 4 state: waiting for i/o request (read thread)
I/O thread 5 state: waiting for i/o request (read thread)
I/O thread 6 state: waiting for i/o request (write thread)
I/O thread 7 state: waiting for i/o request (write thread)
I/O thread 8 state: waiting for i/o request (write thread)
I/O thread 9 state: waiting for i/o request (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
3143 OS file reads, 15089 OS file writes, 4379 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
....
----------------------------
END OF INNODB MONITOR OUTPUT
============================

```

#### 3. purege 线程 
>purge：[pɜːdʒ] v.清除  

事务提交后，使用的undo log可能不在需要，需要purge 线程来回收已经使用并分配的undo页。 purge操作在老版本的innodb中是在master线程中执行的。
  
  下面的例子我们可以看到有**4个**purge线程。
 ```
 mysql> select version() \G;
*************************** 1. row ***************************
version(): 5.7.17
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> show variables like "%purge_thread%";
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_purge_threads | 4     |
+----------------------+-------+
1 row in set (0.01 sec)
 ``` 

#### 4. page cleaner 线程
将老版本的胀页刷新也放到一个线程来做。
mysql> show variables like "%page_clean%";
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_page_cleaners | 1     |
+----------------------+-------+
1 row in set (0.01 sec)


## 2.内存
### 1. 缓冲池
1. 为什么要有缓冲池?
innodb是基于磁盘的存储引擎(有基于内存的引擎)。磁盘的io速度与cpu速度天生就是一个鸿沟。 需要缓冲池来提升读写性能。
innodb在磁盘上的数据是按照**页**来组织的。 在数据库中读取某个也的时候，先将读到的页放入缓冲池中，下一次在读到相同页的时候，直接读取缓冲池的页。 对数据页的修改时候，先修改缓冲池中的页，在以一定的频率刷新到磁盘上。
**缓冲池的大小直接决定了数据库的整体性能**
```shell
mysql> show variables like "%buffer_pool_size%";
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set (0.01 sec)

mysql> 

```
