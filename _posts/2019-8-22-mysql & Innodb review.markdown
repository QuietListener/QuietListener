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
```sql
mysql> show variables like "%page_clean%";
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_page_cleaners | 1     |
+----------------------+-------+
1 row in set (0.01 sec)
```

## 2.内存
### 1. 缓冲池(buffer pool)
#### 1. 为什么要有缓冲池?
innodb是基于磁盘的存储引擎(有基于内存的引擎)。磁盘的io速度与cpu速度天生就是一个鸿沟。 需要缓冲池来提升读写性能。
innodb在磁盘上的数据是按照**页**来组织的。 在数据库中读取某个**页**的时候，先将读到的页放入缓冲池中，下一次在读到相同页的时候，直接读取缓冲池的页。 对数据页的修改时候，先修改缓冲池中的页，在以一定的频率刷新到磁盘上。
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


#### 2. 内存池中存了什么东西？
1. 数据页(date page) , 索引页(index page) 这两个占了缓冲池绝大部分空间。
2. 插入缓冲(insert buffer),自适应哈希索引(adaptive hash index),锁信息(lock info),数据字典信息(data dictionary).
**可以有多个内存池来减少资源竞争，增加并发。**
```sql
mysql> show variables like "%buffer_pool_instances%";
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+
1 row in set (0.07 sec)
```

**公司使用阿里云rds，使用了8个内存池。**

**查看缓冲池的状态**
```shell

mysql> show engine innodb status \G;
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
....
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 7905957
Buffer pool size   8191
Free buffers       1589
Database pages     6574
Old database pages 2428
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 480, not young 1034
0.00 youngs/s, 0.00 non-youngs/s
Pages read 2279, created 4827, written 11466
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 6574, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

```

#### 3. 使用LRU算法管理缓冲池
1. 朴素的LRU算法。
最频繁使用的页在LRU列表的最前端，最少使用的页在LRU列表的最尾端。当缓冲池不能存放新页的时候，淘汰最尾端的页。
2. innodb使用**改进的LRU算法**。
innodb读到的一个新页并不是放到LRU队列的最前端，而是放到LRU长度的37%处，这个位置叫midpoint。
innodb把midpoint之后的列表成为old列表(最不活跃，老了吧)，把midpont之前的列表称为new列表(最活跃，新生事物)，new表里是最热的数据。

![innodb-architect](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-6-4-innodb-lru.jpg)

可以使用innodb_old_blocks_pct来重新设置这个参数。
```sql
mysql> show variables like "innodb_old_blocks_pct";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
1 row in set (0.01 sec)

```

3. 为什么要使用这种midpont改进过LRU算法呢?   

 如果直接将读取到的页放入LRU列表首部，那一些Sql操作可能导致缓冲池的页被淘汰，比如一个慢查询扫描了100万行数据，这些数据所在的页只在这个偶尔出现的慢查询中需要，并不是热点数据。放在首部的话，可能将真正的热点数据淘汰，下一次又要去磁盘读取，降低了效率。

 为了减少上面的情况发生，innodb还加入了一个参数 innodb_old_blocks_time 来控制一个页读取到midpoint位置后还需要等多久才能够放到LRU列表的热端。
 ```sql
 mysql> show variables like "innodb_old_blocks_time";
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
1 row in set (0.02 sec)
 ```

 4. Free List和LRU List 怎么配合
 mysql刚启动的时候LRU list是空的，页都存放在**Free List**中，当需要从缓冲池中分配新的页的时候，Innodb先看看Free List中找，如果有，将页从Free List中删除，添加到LRU List中，如果Free List中也没有空闲页，就会淘汰LRU list尾端的页。当页从old部分移动到new部分时候,此时的发生的操作叫做，**page make young**,因为innodb_old_blocks_time使得页没有从old部分移动到new部分操作叫做**page not made young**。   

使用 show engine innodb status 查看：
```sql
----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 6609174528; in additional pool allocated 0
Dictionary memory allocated 1400366
Buffer pool size   393216  # 缓存池中，总共有393216个页
Free buffers       8191   #Free List的数据
Database pages     384820  # 表示LRU List中的数量
Old database pages 141889
Modified db pages  80 #脏页
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 243955242, not young 0    # page make young 表示 LRU列表中将页移动到前端的数量
14.92 youngs/s, 0.00 non-youngs/s   # page made young 和 not young的每秒次数
Pages read 195807626, created 1410314, written 3981183279
15.24 reads/s, 0.00 creates/s, 260.15 writes/s
Buffer pool hit rate 998 / 1000, young-making rate 2 / 1000 not 0 / 1000 # 缓存命中率，重要指标
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 384820, unzip_LRU len: 0
I/O sum[109776]:cur[2256], unzip sum[0]:cur[0]
```
**一个重要的指标: Buffer pool hit rate 998 / 1000** 表示缓冲池的命中率，一般要高于95%，当前值是99.8%。当小于95%的时候就要查一下是不是有全表扫描污染了LRU列表了。

Innodb1.2之后也可以直接从information_schema中查出来：
```sql
mysql> select * from information_schema.INNODB_BUFFER_POOL_STATS \G;
*************************** 1. row ***************************
                         POOL_ID: 0
                       POOL_SIZE: 8191
                    FREE_BUFFERS: 1534
                  DATABASE_PAGES: 6629
              OLD_DATABASE_PAGES: 2427
         MODIFIED_DATABASE_PAGES: 0
              PENDING_DECOMPRESS: 0
                   PENDING_READS: 0
               PENDING_FLUSH_LRU: 0
              PENDING_FLUSH_LIST: 0
                PAGES_MADE_YOUNG: 483
            PAGES_NOT_MADE_YOUNG: 1034
           PAGES_MADE_YOUNG_RATE: 0
       PAGES_MADE_NOT_YOUNG_RATE: 0
               NUMBER_PAGES_READ: 2279
            NUMBER_PAGES_CREATED: 4882
            NUMBER_PAGES_WRITTEN: 11545
                 PAGES_READ_RATE: 0
               PAGES_CREATE_RATE: 0
              PAGES_WRITTEN_RATE: 0
                NUMBER_PAGES_GET: 5896353
                        HIT_RATE: 0
    YOUNG_MAKE_PER_THOUSAND_GETS: 0
NOT_YOUNG_MAKE_PER_THOUSAND_GETS: 0
         NUMBER_PAGES_READ_AHEAD: 319
       NUMBER_READ_AHEAD_EVICTED: 0
                 READ_AHEAD_RATE: 0
         READ_AHEAD_EVICTED_RATE: 0
                    LRU_IO_TOTAL: 0
                  LRU_IO_CURRENT: 0
                UNCOMPRESS_TOTAL: 0
              UNCOMPRESS_CURRENT: 0
1 row in set (0.01 sec)

ERROR: 
No query specified

mysql> 

```


4. 压缩页   

1.0.x之后支持压缩也，页的大小可以为1KB，2KB，4KB,8KB. 非16KB的页由unzip_LRU 列表来单独管理,比如有8KB，4KB，2KB的unzip_LRU 列表。
例如 show engine innodb status ; 可以看到 

```sql

......
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 384820, unzip_LRU len: 0
I/O sum[109776]:cur[2256], unzip sum[0]:cur[0]
......   

```


Innodb使用**伙伴算法**来分配压缩页: 
**伙伴算法可以参考**: https://blog.csdn.net/csdn_kou/article/details/82355452

例如要申请4KB的页,过程如下
1: 检查unzip_LRU 列表是有有可用的空闲页
2: 如果有直接使用
3: 否则检查8KB的unzip_LRU列表
4: 如果有空闲页，将该页分为2个4KB的页，放到4KB的unzip_LRU列表
5: 如果没有,从LRU列表中申请一个15KB的页，将它分为1个8KB的页，和2个4KB的页，分别放到对应的unzip_LRU列表中。  



5. Flush List  脏页列表   

当LRU列表中的页被修改后，该页就”脏“了，需要将数据写会磁盘。这时可以通过 Innodb的 checkpoint机制将脏页刷新会磁盘。 **Flush List**就是脏页列表。 注意:脏页既存在于LRU list也存在 Flush list中。

可以使用show engine innodb status 查看脏页数量。

```sql
Old database pages 141889
Modified db pages  80 # Flush List中脏页数量
Pending reads 0
```

### 2. 重做日志缓冲(redo log buffer)
Innodb先将重做日志信息放入到这个缓冲区，然后定时刷新到重做日志文件。 一般不用太大，默认8M，可以由innodb_log_buffer_size控制大小。

```shell
mysql> show variables like "innodb_log_buffer_size";
+------------------------+----------+
| Variable_name          | Value    |
+------------------------+----------+
| innodb_log_buffer_size | 16777216 |
+------------------------+----------+
1 row in set (0.10 sec)
mysql> 
```

刷新重做日志到磁盘日子文件的时机:
1. Master Thread 每秒做一次。
2. 每个事务提交后会做一次。
3. 重做日志缓冲区大小小于1/2时候，做一次。

### 3. 额外内存池

### 4. Checkpoint技术

#### 1. 为什么要要有redo log?
1. 当执行update/delete改变了**页**中的记录，**页**就变成了**脏**页，需要将缓冲池中的**脏页**刷新到磁盘~
2. 如果有脏页就刷新到磁盘，开销会非常大~，如果在从缓冲池刷新到磁盘时候宕机，数据就丢失了，不能恢复。为了解决这个问题，防止丢失数据，现在的引擎一般都采用 **Write Ahead Log** 的策略，***当事务提交时候，先写重做日志，再修改页***,当发生宕机的时候可以用重做日志恢复数据~。这就做到了**ACID**的**D**, 重做日志也是先写到**重做日志缓冲**里面。

**注意:重做log里面记录的是页的物理操作，比如在某页偏移800写入0x1111**

#### 2.为什么需要有Checkpoint(检查点)机制?
  1. 如果没有检查点机制? 宕机恢复的时候就需要恢复所有的redo log，量很大速度慢，而且这么多的redo log磁盘也基本存不放下，比如线上的阿里云的rds 我司买的磁盘都是2T的。
  2. 如果有Checkpoint机制，检查点之前的**页** 都已经是刷新到磁盘的了，所以在数据库故障的时候只需要对Checkpoint后面的**重做日志**进行恢复。
     当需页不够用，需要淘汰页的时候，这些淘汰的页需要强制执行Checkpoint，将脏页刷新回磁盘。  

#### 3. 两种Checkpoint: Sharp Checkpoint和Fuzzy Checkpoint
##### 1. Sharp Checkpoint
将所有脏页都刷新回磁盘，一般在关闭数据库的时候使用。参数innodb_fast_shutdown=1
##### 2. Fuzzy Checkpoint
只刷回一部分脏页，有几种情况:
a. Master Thread每秒或者每n秒从缓冲池中将脏页列表中刷新一定比例到磁盘。       

b. Innodb要保证一定比例的空闲页比如(1000个)可供使用，当没有这么多的时候，就需要溢出尾端的页，如果有脏页，就会执行Checkpoint,比如下面表示至少需要1024个空闲页。  


```shell
mysql> show variables like "%scan_depth";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_lru_scan_depth | 1024  |
+-----------------------+-------+
1 row in set (0.01 sec) 
```  

c. 如果重做日志已经满了，不可用，需要强制做checkpoint。  重做日志是固定大小循环使用的。 
  

d.**Dirty Page Too Much** 
    当脏页太多超过一定比例会强制做checkpoint,让脏页保持在一定比例一下。


```shell
mysql> show variables like "%dirty_pages_pct";
+----------------------------+-----------+
| Variable_name              | Value     |
+----------------------------+-----------+
| innodb_max_dirty_pages_pct | 75.000000 |
+----------------------------+-----------+
1 row in set (0.01 sec)
```


     
### 3. Innodb关键特性
#### 1. 插入缓冲(Insert Buffer)

##### 1. Innodb索引是什么样的?
 1. innodb使用B+树来实现索引。 
 2. 索引分为**聚集索引(Primary Key)**和**辅助索引(Secondary index)**
 3. 聚集索引是按照id递增顺序存放在一起放在数据页(挨着放的，像是差不多大的primary key聚集在一起),所以聚集索引的顺序插入和顺序读取都是非常快的(磁盘的随机写入/读取都很慢)。
 4. 辅助索引的值并不是数据本身，而是聚集索引(primary key)。所以如果按照辅助索引查找数据，分为两步，1. 通过辅助索引找primay key，2. 通过primary key找到数据。

 ##### 2. 为什么需要Insert Buffer。   

 当插入一条数据时候，插入聚集索引非常快(顺序插入到一个数据页，有可能数据页还是在Buffer Pool,还是热的)，但是辅助索引是离散的，很有可能辅助索引所在的页没有在内存，需要从磁盘读入。如果要同时插入会大大影响性能。
 比如表

 ```sql
 create table t (
   a int auto_increment,
   b varchar(10),
   primary key(a),
   key(b)
 );
```   

当插入 insert into t values(NULL, "abc");  主键自增，插入到一个页A中，辅助索引插入到页B中。
当我们再插入一条 insert into t values(NULL, "12$"); 主键自增，我们有非常大的可能 primary key也会插入到一个页A中,因为是顺序插入，而辅助索引插很可能入到页F中。因为辅助索引不是顺序的( "abc"和”12$“可能中间隔了10000个节点)。

**所以引入insert buffer** 对辅助索引的插入或者更新，不是每次都插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，如果在直接插入，如果没有放入到**insert buffer**。 然后以一定的比例和情况将insert buffer和辅助索引节点合并，而且可以多个操作合为一个，加快速度。

##### 3. 使用 insert buffer的时机
1. 索引是辅助索引，  
2. 索引不是唯一索引。 
不是唯一索引因为 **唯一索引在插入时候要判断索引的唯一性,如果重复需要立即失败。所以唯一索引会减慢插入速度**

所以**insert buffer** 解决的是性能问题


#### 2. 两次写(double write)
##### 1. 为什么有两次写?
当在将某个也写入某个页的表中(磁盘)，只写了一部分，比如16K的页的数据，只写了6K系统就挂了，所以这个页就被写坏了。这时候使用redo log也恢复不了。因为redo log存的是  ”在某页偏移800写入0x1111“这种数据，如果这个”某页“都被写坏了，就恢复不了了。所以,需要引入 **"两次写"** 来解决这个问题。


所以: **两次写带来的是可靠性的提升.**

##### 2. 两次写的原理。
###### 1. 先使用memcopy先将脏页放入double write buffer(2M). 
这一步使用的是速度很快。因为是内存拷贝
###### 2. 将double write buffer 分两次每次1M将double write buffer 顺序写入共享表空间中的物理磁盘(2M)，马上调fsync保证写盘。
这一步写磁盘，但是是顺序写，也很快
###### 3. 完成1，2之后，把脏页写入到磁盘中，这个时候就不是顺序写了。


![innodb-architect](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/2020-06-06-innodb-double-write.jpg)

使用:
**show GLOBAL  status like "%dblwr%"** 查看状态线上库

```java
Innodb_dblwr_page_number: 4521032878
Innodb_dblwr_writes:   211745660
```
4521032878/211745660 = 21 说明服务器比较忙


#### 3. 自适应哈希AHI(adaptive hash index)
innodb默认使用B+树来索引，一般需要查3到4次才能查到数据，hash基本是1次就能查到，如果频繁查询某个热点数据，innod不会建立hash索引，加快速度。

自适应哈希只能 在 等值查询才能使用，范围查询不能用。
比如: WHERE a=xxx and b=xxx。
```shell
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 11723, seg size 11725, 18697212 merges
merged operations:
 insert 18553004, delete mark 4516929, delete 4326944
discarded operations:
 insert 0, delete mark 0, delete 0
AHI PARTITION 1: Hash table size 1593833, node heap has 2 buffer(s)
AHI PARTITION 2: Hash table size 1593833, node heap has 1 buffer(s)
AHI PARTITION 3: Hash table size 1593833, node heap has 1 buffer(s)
AHI PARTITION 4: Hash table size 1593833, node heap has 1 buffer(s)
AHI PARTITION 5: Hash table size 1593833, node heap has 9 buffer(s)
AHI PARTITION 6: Hash table size 1593833, node heap has 186 buffer(s)
AHI PARTITION 7: Hash table size 1593833, node heap has 3 buffer(s)
AHI PARTITION 8: Hash table size 1593833, node heap has 2 buffer(s)
14.76 hash searches/s, 393.23 non-hash searches/s #可以看到 hash search和no-hash search
```


## 1. 启动，恢复和恢复
关闭时 innodb_fast_shutdown 影响着表的存储引擎~ 
### 1. innodb_fast_shutdown为0:
mysql关闭时，InnoDb需要完成所有full purge和merge insert buffer,并且将所有脏页刷新会磁盘。 这个需要很多时间极端时候可能是几小时。
### 2. innodb_fast_shutdown为1:
上面的操作不必做完，但是缓冲池中的脏数据需要刷新会磁盘
### 3. innodb_fast_shutdown为2:
上面1和2都不用做，但是日子都要写入日志文件，事物不会丢失，下次mysql启动会进行恢复操作。


## 3.文件
 mysql中有几类文件:
 1. 参数文件: my.cnf
 2. 日志文件: 错误日志，二进制文件(bin log)，慢查询文件，查询日志文件
 3. pid
 4. 表结构文件:存放mysql表结构定义
 5. 存储引擎相关文件。innodb的redo log和 和 undo log.


### 1. 参数文件
一般放在 my.cnf

#### 1. 查看参数
可以使用 show variables 查看所有参数，也可使用 show variables like "%xxx%" 来过滤。
```sql
mysql> show variables like "%buffer_pool%";
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 134217728      |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 25             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 1              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_size             | 134217728      |
+-------------------------------------+----------------+
10 rows in set (0.06 sec)

mysql> 

```
#### 2. 参数类型
1. 动态 在运行期间可以修改
2. 静态 在运行期间不可以修改

**动态参数设置:**
可以分为 global和session两部分。

例如:
```sql
mysql> select @@session.autocommit ;
+----------------------+
| @@session.autocommit |
+----------------------+
|                    1 |
+----------------------+
1 row in set (0.01 sec)

mysql> select @@global.autocommit ;
+---------------------+
| @@global.autocommit |
+---------------------+
|                   1 |
+---------------------+
1 row in set (0.00 sec)

```

设置 自动提交参数:
```sql
mysql> set @@session.autocommit = 0;
Query OK, 0 rows affected (0.03 sec)

mysql> select @@session.autocommit ;
+----------------------+
| @@session.autocommit |
+----------------------+
|                    0 |
+----------------------+
1 row in set (0.00 sec)
```





### 2. 错误文件
```sql
mysql> show variables like "%log_error%";
+---------------------+-----------------------------------------------------------+
| Variable_name       | Value                                                     |
+---------------------+-----------------------------------------------------------+
| binlog_error_action | ABORT_SERVER                                              |
| log_error           | /usr/local/mysql/data/junjunkaifarnxiaojuyuwang.local.err |
| log_error_verbosity | 3                                                         |
+---------------------+-----------------------------------------------------------+
3 rows in set (0.02 sec)

```

**查看一下 /usr/local/mysql/data/junjunkaifarnxiaojuyuwang.local.err**
连链接断开都记录在error中。

```shell
➜  test-thrift-client git:(master) tail  /usr/local/mysql/data/junjunkaifarnxiaojuyuwang.local.err
2020-06-06T22:31:17.825828Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 27396520ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-07T01:25:36.408242Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 10492072ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-07T04:11:25.926892Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 9915267ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-08T01:37:45.824116Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 77150274ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-08T01:46:55.215325Z 352 [Note] Aborted connection 352 to db: 'information_schema' user: 'root' host: 'localhost' (Got timeout reading communication packets)
2020-06-08T22:33:08.826486Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 43299187ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-09T01:33:15.473189Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 10800941ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-09T01:35:07.402265Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 110309ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-06-09T02:18:47.650939Z 319 [Note] Aborted connection 319 to db: 'unconnected' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-06-09T02:18:47.650804Z 339 [Note] Aborted connection 339 to db: 'stock' user: 'root' host: 'localhost' (Got an error reading communication packets)
```


### 3. 慢查询文件
1. 阈值
```sql
mysql> show variables like "long_query_time"
    -> ;
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
1 row in set (0.01 sec)

```
2. 是否开启
```sql
mysql> show variables like "%slow_query_log%";
+---------------------+----------------------------------------------------------+
| Variable_name       | Value                                                    |
+---------------------+----------------------------------------------------------+
| slow_query_log      | OFF                                                      |
| slow_query_log_file | /usr/local/mysql/data/junjunkaifarnxiaojuyuwang-slow.log |
+---------------------+----------------------------------------------------------+
2 rows in set (0.02 sec)

mysql> 
```

**如果没有使用索引的sql也计入slow log**
```sql
mysql> show variables like "%log_queries_not%";
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| log_queries_not_using_indexes | OFF   |
+-------------------------------+-------+
1 row in set (0.02 sec)

mysql> 

```

**5.1开始可以从 mysql.slow_log 表里查询了:**
```sql
mysql> show create table  mysql.slow_log \G;
*************************** 1. row ***************************
       Table: slow_log
Create Table: CREATE TABLE `slow_log` (
  `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `query_time` time(6) NOT NULL,
  `lock_time` time(6) NOT NULL,
  `rows_sent` int(11) NOT NULL,
  `rows_examined` int(11) NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int(11) NOT NULL,
  `insert_id` int(11) NOT NULL,
  `server_id` int(10) unsigned NOT NULL,
  `sql_text` mediumblob NOT NULL,
  `thread_id` bigint(21) unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> 

```

如果要从表里查slow log的话需要将下面参数设置为TABLE; 当前是FILE

```sql
mysql> show variables like "log_output"
    -> ;
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+
1 row in set (0.02 sec)

mysql> 
```

**打开慢日志，并执行一条很久的sql**
```sql
mysql> set @@global.slow_query_log = on;
Query OK, 0 rows affected (0.00 sec)

mysql> 
mysql> select sleep(11)\G; #睡11秒
*************************** 1. row ***************************
sleep(11): 0
1 row in set (11.01 sec)

ERROR: 
No query specified

mysql> 

```

**查看slow log；**

```log
➜  test-thrift-client git:(master) tail /usr/local/mysql/data/junjunkaifarnxiaojuyuwang-slow.log
/usr/local/mysql/bin/mysqld, Version: 5.7.17 (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /tmp/mysql.sock
Time                 Id Command    Argument
# Time: 2020-06-09T03:24:38.308863Z
# User@Host: root[root] @ localhost []  Id:   353
# Query_time: 11.010569  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
use information_schema;
SET timestamp=1591673078;
select sleep(11);
➜  test-thrift-client git:(master) 
```


可以使用 **mysqldumpslow** 来分析 slow log
```sql
➜  test-thrift-client git:(master) mysqldumpslow   /usr/local/mysql/data/junjunkaifarnxiaojuyuwang-slow.log

Reading mysql slow query log from /usr/local/mysql/data/junjunkaifarnxiaojuyuwang-slow.log
Count: 1  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=0.0 (0), 0users@0hosts
  # Time: N-N-09T03:N:N.308863Z
  # User@Host: root[root] @ localhost []  Id:   N
  # Query_time: N.N  Lock_time: N.N Rows_sent: N  Rows_examined: N
  use information_schema;
  SET timestamp=N;
  select sleep(N)

➜  test-thrift-client git:(master) 

```


### 4. 查询日志
查询日志记录了所有对mysql数据库请求的信息，无论这些请求是否得到了正确的执行。

```sql
mysql> show variables like "%general_log%";
+------------------+-----------------------------------+
| Variable_name    | Value                             |
+------------------+-----------------------------------+
| general_log      | ON                                |
| general_log_file | /usr/local/mysql/data/tyj-dev.log |
+------------------+-----------------------------------+
2 rows in set (0.00 sec)
```

```shell
➜  ~ tail  /usr/local/mysql/data/tyj-dev.log                  
/usr/local/mysql/bin/mysqld, Version: 5.7.17 (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /tmp/mysql.sock
Time                 Id Command    Argument
2020-06-11T07:08:21.111679Z	    5 Query	show variables like "%general_log%"
2020-06-11T07:09:21.520872Z	    5 Query	select 1

```

### 5. 二进制文件(binlog)
二进制文件记录了对mysql数据库执行更改的操作，select和show操作不会计入这个log。
配置my.cnf
```shell
#bin log
log_bin=ON
server-id=1001
```
**mysql5.7以后必须有server-id**，不然启动报错，这个id不能和集群中其他机器的id相同。


#### 1. 使用 **show master status**查看当前binlog的状态。

```sql
mysql> show master status;
+-----------+----------+--------------+------------------+-------------------+
| File      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-----------+----------+--------------+------------------+-------------------+
| ON.000002 |     2159 |              |                  |                   |
+-----------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> insert into test values (NULL,"b");
Query OK, 1 row affected (0.02 sec)

mysql> show master status;
+-----------+----------+--------------+------------------+-------------------+
| File      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-----------+----------+--------------+------------------+-------------------+
| ON.000002 |     2432 |              |                  |                   |
+-----------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> select * from test;
+----+------+
| id | name |
+----+------+
|  1 | a    |
|  2 | b    |
+----+------+
2 rows in set (0.00 sec)

mysql> show master status;
+-----------+----------+--------------+------------------+-------------------+
| File      | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-----------+----------+--------------+------------------+-------------------+
| ON.000002 |     2432 |              |                  |                   |
+-----------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> 

```


当前使用的binlog是**ON.000002**，当使用 **insert into test values (NULL,"b");** 插入一条记录后Position从2159变为2432,说明binlog变化了，执行select后Position保持2432不变。说明select操作没有记如binlog。


**还可以使用show binlog events in "ON.000002"来查看具体每一次的变化。**

```sql
mysql> show binlog events in "ON.000002";
+-----------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Log_name  | Pos  | Event_type     | Server_id | End_log_pos | Info                                                                                                                                                                          |
+-----------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ON.000002 |    4 | Format_desc    |      1001 |         123 | Server ver: 5.7.17-log, Binlog ver: 4                                                                                                                                         |
| ON.000002 |  123 | Previous_gtids |      1001 |         154 |                                                                                                                                                                               |
| ON.000002 |  154 | Anonymous_Gtid |      1001 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 |  219 | Query          |      1001 |         300 | BEGIN                                                                                                                                                                         |
| ON.000002 |  300 | Table_map      |      1001 |         356 | table_id: 377 (stock.stock)                                                                                                                                                   |
| ON.000002 |  356 | Update_rows    |      1001 |         462 | table_id: 377 flags: STMT_END_F                                                                                                                                               |
| ON.000002 |  462 | Xid            |      1001 |         493 | COMMIT /* xid=163 */                                                                                                                                                          |
| ON.000002 |  493 | Anonymous_Gtid |      1001 |         558 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 |  558 | Query          |      1001 |         639 | BEGIN                                                                                                                                                                         |
| ON.000002 |  639 | Table_map      |      1001 |         695 | table_id: 377 (stock.stock)                                                                                                                                                   |
| ON.000002 |  695 | Update_rows    |      1001 |         801 | table_id: 377 flags: STMT_END_F                                                                                                                                               |
| ON.000002 |  801 | Xid            |      1001 |         832 | COMMIT /* xid=176 */                                                                                                                                                          |
| ON.000002 |  832 | Anonymous_Gtid |      1001 |         897 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 |  897 | Query          |      1001 |         978 | BEGIN                                                                                                                                                                         |
| ON.000002 |  978 | Table_map      |      1001 |        1034 | table_id: 377 (stock.stock)                                                                                                                                                   |
| ON.000002 | 1034 | Update_rows    |      1001 |        1140 | table_id: 377 flags: STMT_END_F                                                                                                                                               |
| ON.000002 | 1140 | Xid            |      1001 |        1171 | COMMIT /* xid=182 */                                                                                                                                                          |
| ON.000002 | 1171 | Anonymous_Gtid |      1001 |        1236 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 | 1236 | Query          |      1001 |        1348 | create database mysqllearn                                                                                                                                                    |
| ON.000002 | 1348 | Anonymous_Gtid |      1001 |        1413 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 | 1413 | Query          |      1001 |        1580 | use `stock`; create table test ( `id` int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY, `name` varchar(255) )                                                                      |
| ON.000002 | 1580 | Anonymous_Gtid |      1001 |        1645 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 | 1645 | Query          |      1001 |        1886 | use `mysqllearn`; CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 |
| ON.000002 | 1886 | Anonymous_Gtid |      1001 |        1951 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 | 1951 | Query          |      1001 |        2029 | BEGIN                                                                                                                                                                         |
| ON.000002 | 2029 | Table_map      |      1001 |        2085 | table_id: 745 (mysqllearn.test)                                                                                                                                               |
| ON.000002 | 2085 | Write_rows     |      1001 |        2128 | table_id: 745 flags: STMT_END_F                                                                                                                                               |
| ON.000002 | 2128 | Xid            |      1001 |        2159 | COMMIT /* xid=205 */                                                                                                                                                          |
| ON.000002 | 2159 | Anonymous_Gtid |      1001 |        2224 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                          |
| ON.000002 | 2224 | Query          |      1001 |        2302 | BEGIN                                                                                                                                                                         |
| ON.000002 | 2302 | Table_map      |      1001 |        2358 | table_id: 745 (mysqllearn.test)                                                                                                                                               |
| ON.000002 | 2358 | Write_rows     |      1001 |        2401 | table_id: 745 flags: STMT_END_F                                                                                                                                               |
| ON.000002 | 2401 | Xid            |      1001 |        2432 | COMMIT /* xid=207 */                                                                                                                                                          |
+-----------+------+----------------+-----------+-------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
33 rows in set (0.00 sec)


```

#### 2. 二进制文件的作用
1. 恢复:使用binlog进行恢复数据
2. 复制:从mysql与主mysql进行实时同步
3. 审计: 判断数据是否有注入攻击等

#### 3. 相关参数:
```sql
mysql> show variables like "%log_bin%";
+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /usr/local/mysql/data/ON       |
| log_bin_index                   | /usr/local/mysql/data/ON.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
6 rows in set (0.00 sec)
```
log_bin = xxname 用来开启binlog，**开启后会有1%的性能损失**
这里我们可看到 **log_bin_basename** 就是具体的binlog文件。Binlog的后缀后不断增加。这里有两个ON.000001，ON.000002。   
**log_bin_index** 是binlog的索引文件，用来记录过往产生过的二进制日志序号。

```sql
mysql> system ls -l /usr/local/mysql/data/ON*
-rw-r-----  1 junjun  _mysql   177 Jun 11 15:50 /usr/local/mysql/data/ON.000001
-rw-r-----  1 junjun  _mysql  2432 Jun 11 16:16 /usr/local/mysql/data/ON.000002
-rw-r-----  1 junjun  _mysql    24 Jun 11 15:50 /usr/local/mysql/data/ON.index
mysql> 

```

```sql
mysql> show variables like "%binlog%";
+-----------------------------------------+----------------------+
| Variable_name                           | Value                |
+-----------------------------------------+----------------------+
| binlog_cache_size                       | 32768                |
| binlog_format                           | ROW                  |
| max_binlog_cache_size                   | 18446744073709547520 |
| max_binlog_size                         | 1073741824           |
| sync_binlog                             | 1                    |
+-----------------------------------------+----------------------+
20 rows in set (0.00 sec)

```

##### 1. max_binlog_size:
   binlog的大小默认为1G
##### 2. binlog_cache_size:
   默认为32K，这个参数是基于会话的，每一个会话会分配一个。  
   使用事务的引擎(Innodb)，会将未提交的二进制日志写入一个缓存中，**当commit的时候才写入binlog文件**, 如果binlog_cache_size设置太小cache放不下，会写入磁盘零时文件，Binlog_cache_disk_use会显示使用磁盘临时文件的次数。Binlog_cache_use会显示使用binlog_cache的次数。
   下面是一个线上的一个结果：
   
**show global status like "%binlog_cache%";**，一共用了1699305934次binlog缓存，15次磁盘临时文件，说明binlog_cache_size大小比较合适。

```sql
Binlog_cache_disk_use 15
Binlog_cache_use 1699305934

```

##### 3. **binlog_format**二进制日志的格式
   这个参数控制二进制文件的日志。非常重要。
   在MySQL有3种格式:
1. statement   
记录的是逻辑SQL语句，比如update test set name="11".这条sql修改了100万行，binlog也只记录这一个sql语句。

2. row   
记录的是每一行的改变，比如update test set name="11".这条sql修改了100万行，那binlog会记录100万个行改变的记录，可能binlog会增加几十兆

3. mixed
默认使用statement来记录，有一些情况使用row格式，
a. 使用了UUID，USER,CURRENT_USER,FOUND_ROW,ROW_COUNT等不确定函数。
b. 使用了零时表
c. 使用自定义函数
d. 使用insert delay


**通常使用binlog_format=row 在数据库恢复和复制的时候会更可靠**，缺点是binlog会更大，master和slave之间同步更慢。



#### 4. 查看binlog
binlog不可读，可以使用mysqlbinlog 来查看 binlog。
```sql
➜  data mysqlbinlog -vv --start-position=2224  ON.000002
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#200611 15:50:58 server id 1001  end_log_pos 123 CRC32 0x1fd23855 	Start: binlog v 4, server v 5.7.17-log created 200611 15:50:58 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
YuLhXg/pAwAAdwAAAHsAAAABAAQANS43LjE3LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAABi4uFeEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AVU40h8=
'/*!*/;
# at 2224
#200611 16:16:38 server id 1001  end_log_pos 2302 CRC32 0x25f73241 	Query	thread_id=4	exec_time=0	error_code=0
SET TIMESTAMP=1591863398/*!*/;
SET @@session.pseudo_thread_id=4/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 2302
#200611 16:16:38 server id 1001  end_log_pos 2358 CRC32 0xb8cbc9a7 	Table_map: `mysqllearn`.`test` mapped to number 745
# at 2358
#200611 16:16:38 server id 1001  end_log_pos 2401 CRC32 0xac0133e9 	Write_rows: table id 745 flags: STMT_END_F

BINLOG '
ZujhXhPpAwAAOAAAADYJAAAAAOkCAAAAAAEACm15c3FsbGVhcm4ABHRlc3QAAgMPAv0CAqfJy7g=
ZujhXh7pAwAAKwAAAGEJAAAAAOkCAAAAAAEAAgAC//wCAAAAAQBi6TMBrA==
'/*!*/;
### INSERT INTO `mysqllearn`.`test`
### SET
###   @1=2 /* INT meta=0 nullable=0 is_null=0 */
###   @2='b' /* VARSTRING(765) meta=765 nullable=1 is_null=0 */
# at 2401
#200611 16:16:38 server id 1001  end_log_pos 2432 CRC32 0x6a28a159 	Xid = 207
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
➜  data 

```


### 6. mysql表结构(与引擎无关)
msyql都有一个后缀为frm的文件，这个文件记录了该表的表结构定义。
例如数据库mysqllearn下有一个test表，在mysqllearn目录下生成了一个test.frm文件。

```sql
mysql> desc test;
+-------+--------------+------+-----+---------+----------------+
| Field | Type         | Null | Key | Default | Extra          |
+-------+--------------+------+-----+---------+----------------+
| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255) | YES  |     | NULL    |                |
+-------+--------------+------+-----+---------+----------------+
2 rows in set (0.01 sec)

```
文件信息

```shell
➜  mysqllearn pwd
/usr/local/mysql/data/mysqllearn
➜  mysqllearn ls -lahs                          
total 224
  0 drwxr-x---   5 junjun  _mysql   160B Jun 11 16:15 .
  0 drwxr-x---  73 junjun  _mysql   2.3K Jun 11 16:12 ..
  8 -rw-r-----   1 junjun  _mysql    65B Jun 11 16:12 db.opt
 24 -rw-r-----   1 junjun  _mysql   8.4K Jun 11 16:15 test.frm
192 -rw-r-----   1 junjun  _mysql    96K Jun 11 16:16 test.ibd

```

### 7. Innodb存储引擎文件
#### 1. 表空间文件
Innodb存储的数据默然按照表空间来存放: 默认情况下会有一个名字为ibdata1的文件作为表空间文件：

```sql
mysql> show variables like "%innodb_data_file%";
+-----------------------+------------------------+
| Variable_name         | Value                  |
+-----------------------+------------------------+
| innodb_data_file_path | ibdata1:12M:autoextend |
+-----------------------+------------------------+
1 row in set (0.01 sec)

```

是嘛可以看到 **ibdata1:12M:autoextend** 表示，表空间文件为ibdata1，默认为12M，autoextend表示可以自动增加。 我们可以从下面看到 ibdata1 已经76M了。
```shell
➜  data pwd
/usr/local/mysql/data
➜  data ls -lahs ibdata*
155648 -rw-r-----  1 junjun  _mysql    76M Jun 11 16:16 ibdata1
```

**每个表一个表空间文件**
如果设置了 innodb_file_per_table 了,每个表都会有一个表空间文件

```sql
mysql> show variables like "%per_table%";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
1 row in set (0.00 sec)
```

下面**test.ibd**就是test表的表空间文件。但是这个单独的表空间文件，仅仅存放，该表的数据，索引，插入缓冲bitmap等，其余信息还是在默认表空间中。

```shell
➜  mysqllearn pwd
/usr/local/mysql/data/mysqllearn
➜  mysqllearn ls -lahs                          
total 224
  0 drwxr-x---   5 junjun  _mysql   160B Jun 11 16:15 .
  0 drwxr-x---  73 junjun  _mysql   2.3K Jun 11 16:12 ..
  8 -rw-r-----   1 junjun  _mysql    65B Jun 11 16:12 db.opt
 24 -rw-r-----   1 junjun  _mysql   8.4K Jun 11 16:15 test.frm
192 -rw-r-----   1 junjun  _mysql    96K Jun 11 16:16 test.ibd

```

#### 2. 重做日志文件
##### 1. 作用
重做日志记录了innodb存储引擎的事务日志，当掉电后可以恢复到掉电钱的时刻。

默认有1个重做日志组，包括2个重做日志，如下所示。采用**循环写**的方式写日志，日志1写满后，会写日志2，日志2写满后会再切换到日志1.

**下面两个就是重做日志文件:**
```sql
mysql> system ls -l /usr/local/mysql/data/ib_log*;
-rw-r-----  1 junjun  _mysql  50331648 Jun 11 16:16 /usr/local/mysql/data/ib_logfile0
-rw-r-----  1 junjun  _mysql  50331648 Jun 11 16:16 /usr/local/mysql/data/ib_logfile1
```

##### 2.重要的参数:
```sql
mysql> show variables like "innodb_log%";
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| innodb_log_buffer_size      | 16777216 |
| innodb_log_checksums        | ON       |
| innodb_log_compressed_pages | ON       |
| innodb_log_file_size        | 50331648 |
| innodb_log_files_in_group   | 2        |
| innodb_log_group_home_dir   | ./       |
| innodb_log_write_ahead_size | 8192     |
+-----------------------------+----------+
7 rows in set (0.01 sec)
```
1. innodb_log_file_size   
每个重做日志的大小 默认为 512M

2. innodb_log_files_in_group   
每一个日志组里日志文件个数，默认为2

3. innodb_log_group_home_dir 
日志文件所在目录 

##### 3. tip
重做日志文件大小对性能有很大的影响，设置得太大，回复时可能需要很长时间，日志太小，可能导致一个事务多次切换重做日志，日志太小的话，**会让async checkpoint更频繁**，降低性能。

##### 4. binlog和redo log的区别。
1. binlog 记录的是mysql的日志，跟引擎无关； redo log记录的是innodb的事务日志。
2. binlog 不管是 statement还是row，都世纪路的一个事务的具体操作，比如对数据表的更改，对某几行的更改。
   redo log是记录的每一个物理页的修改。例如:把页A偏移量为0x111的位置修改为0xab12.

##### 5.  写重做日志
重做日志也是先写入到 redo log buffer中，然后再将重做日志缓存中的内容写入磁盘。 
有几个时机写重做日志:其中有个参数控制在commit时候写日志的行为.

```sql
mysql> show variables like "innodb_flush_log_at_tr%";
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
1 row in set (0.00 sec)

```
**innodb_flush_log_at_trx_commit**有0，1，2三个值，0表示commit时候不将事务的重做日志同步到磁盘。
1表示写入磁盘，并且调用fsync(保证确实写入磁盘)，2表示写入磁盘，不调用fsync(数据可能还在文件系统的缓存中)。 所以要保证ACID的D，持久性就必须把**innodb_flush_log_at_trx_commit**设置为1.




## 4.表

### 1 索引组织表
Innodb是根据主键顺序组织存放的，在Innodb中每一个表都**必须**有一个主键(Primary Key),如果没有明确指定主键，Innodb会按下面的方式来创建主键:

1. 如果有**非空唯一索引(Unique NOT NULL)**,如果有，选为主键。
   如果有多个这样的非空唯一索引，选建表时第一个定义的为主键。  

2. 不符合上面条件 ，自动生成一个6字节大小的指针为主键。

### 2 Innodb逻辑存储结构

#### 1. 表空间
所有数据都被逻辑存放在一个空间中，叫做表空间。表空间由**段(segment)**,**区(extent)**, **页(page)**组成。

 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200620-mysql-innodb_.jpg)

 Innodb有一个共享表空间文件，ibdata1, 如果配置了inndb_file_per_table=ON,每个表都有一个单独的表空间文件，但是这个单独的表空间文件只存放数据，索引和插入缓冲bitmap，其他的比如回滚信息(undo)，插入缓存索引页，系统事务信息，二次写缓冲都还在共享表空间中。

#### 2. 段(segment)   
表空间由各个段组成，比如数据段，索引段，回滚段等。 Innodb表由索引组织的，那么**数据段就是B+树的叶子节点**（leaf node segment）,**索引段为B+树的非索引节点**(no-leaf node segment)


#### 3. 区(extent)     
区是有连续页组成的空间，每个区大小固定为1M。

#### 4. 页(块) 
页是Innodb磁盘管理的最小单位    
**常见的页有:** 数据页(B-tree Node) , undo页(undo log page), 插入缓冲空间列表页(Insert Buffer free list),未压缩的二进制大对象页(uncompressed Blob page), 压缩的二进制大对象页(uncompressed Blob page)，事务数据页(Transaction System page)。

#### 5.行(row)
Innodb是面向行的数据库(row-oriented)，也就是说数据是按行来进行存放的，每页(16K)存放的记录也是有规定的,只能存放 7992行(16K/2-200)记录。
##### 面向列的数据库vs面向行的数据库 
如果面向行的数据库有100个column，如果值需要其中一个column的数据，也需要将所有的100column读出来，然后修改。  面向列的数据就可以值取这个column的数据即可。比如我只想分析性别和年龄两个column时候，面向列的数据库就更有优势。 所以面向列的数据库多用子啊olap中。






# 5. 索引与算法
## 1. B+树 
这里的B是Balance的意思, 
B+树的飞叶子几点是不存放数据的，所有数据都存放在叶子节点，叶子节点之间用双向链表**按顺序**链接起来~

看这里: https://www.cnblogs.com/nullzx/p/8729425.html

## 2. B+树索引
Innodb的B+树索引每一个飞叶子节点很大，所有基本上B+树只有2到4层，也就是最多需要2到4次io就能完成B+树的搜索，如果磁盘一秒可以有100到200次io，所以2到4次io也就话20毫秒左右的时间。 非常快了~

### 1. 聚集索引（clustered index）
####  理解聚集索引:       
1. innodb表中的数据按照主键顺序存放，
2. 聚集索引就是取表中主键建立的B+树
3. 叶子节点存放着所有**数据页** ( **叶子节点是页，不是一行一行的数据**哈)，数据页之间通过双向链表来进行链接。
4. 非叶子几点存放的是**主键的值以及指向数据页的偏移**。
5. 关于顺序存放，数据按主键**顺序存放**,这里的**顺序**不是物理上的顺序，是逻辑上的顺序，第一:数据页不是物理上连续的，是用双向链表来链接的。 第二：每个数据页上的数据也是按照双向链表来链接的。 所以都是**逻辑上的顺序**


**每张表只能一个主键，只能有一个聚集索引**
下面是一个聚集索引的例子:   
数据页有3个，用双向链表链接，非数据页只有一个，存放的是主键和指向相应数据页的偏移(pointer)。   

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200622mysql-innodb-bplustree.jpg)


### 2. 辅助索引 （secondary index）
辅助索引也叫做 **非聚集索引**，叶子节不包含行记录的全部数据，叶子节点包含**主键的值**，还包括一个书签(bookmark)，书签高数Innodb到哪里去找到与索引相应的行数据。


#### 1. 使用辅助索引来查询数据的过程：
(假设辅助索引和聚集索引的B+树都是3层)
假设表为t，主键为id，辅助索引为name
select * from t where name = "Tom"   

1. 使用辅助索引name来找到对应的主键id，这里需要3次io，因为辅助索引为3层。  
2. 再按照聚集索引查找具体的数据页，io也为3层。一共需要6次io。   

**下面是一个辅助索引查询的示意图**     
可以看到辅助索引只有一个数据页，辅助索引仲存放的key是辅助索引的值，如name，pointer是主键。

![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200622-mysql-innodb-secondarIndex.jpg)


### 3. 索引分裂

### 4. 索引管理

#### 1.  索引的添加删除    

1. 加索引
alter table xxTableName add index/key xxxIndexName index type (columnName1,columnName2,...) 

2. 删索引
alter table xxTableName drop index/key xxxIndexName.


#### 1. 创建例子
1. 下面创建了一个联合索引 index_name_user_id，一个index_info值索引了info的钱5个字符。   
2. Sub_part指定是不是部分索引。
3. **cardinality** 非常重要，表中唯一值的个数，
> cardinality /kɑːdɪ'nælɪtɪ/ n.基数
优化器会使用这个值来决定是否使用这个索引，所以很重要，这个值并不是实时更新的，不是每次索引更新都会更新，要强制清晰的话可以使用analyze table命令。   

**如果有时候建立索在explain的时候并没有使用，或者两条差不多的语句一条使用索引，一个使用全表扫描，可以使用analyze table一下**   
**可以在高峰期使用analyze table，这能使得优化器更好的工作**    


```sql
mysql> alter table test add index index_info (info(5));
Query OK, 0 rows affected (0.14 sec)
Records: 0  Duplicates: 0  Warnings: 0
mysql> alter table test add index index_name_user_id (name,user_id);
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show create table test;
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                                                                                                                                                       |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test  | CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `user_id` int(11) DEFAULT NULL,
  `info` varchar(50) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `key_1` (`user_id`,`name`),
  KEY `index_info` (`info`(5)),
  KEY `index_name_user_id` (`name`,`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 |
+-------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> show index from test;
+-------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table | Non_unique | Key_name           | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| test  |          0 | PRIMARY            |            1 | id          | A         |           3 |     NULL | NULL   |      | BTREE      |         |               |
| test  |          1 | key_1              |            1 | user_id     | A         |           3 |     NULL | NULL   | YES  | BTREE      |         |               |
| test  |          1 | key_1              |            2 | name        | A         |           3 |     NULL | NULL   | YES  | BTREE      |         |               |
| test  |          1 | index_info         |            1 | info        | A         |           3 |        5 | NULL   | YES  | BTREE      |         |               |
| test  |          1 | index_name_user_id |            1 | name        | A         |           3 |     NULL | NULL   | YES  | BTREE      |         |               |
| test  |          1 | index_name_user_id |            2 | user_id     | A         |           3 |     NULL | NULL   | YES  | BTREE      |         |               |
+-------+------------+--------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
6 rows in set (0.00 sec)
```

#### 2. 几个术语 DDL DML
```sql
TCL （Transaction Control Language）：事务控制语言   
DML（data manipulation language）： 它们是SELECT、UPDATE、INSERT、DELETE，这4条命令是用来对数据库里的数据进行操作的语言     

DDL（data definition language）： DDL比DML要多，主要的命令有CREATE、ALTER、DROP等，DDL主要是用在定义或改变表（TABLE）的结构，数据类型，表之间的链接 和约束等初始化工作上，他们大多在建立表时使用 
 
DCL（Data Control Language）：  是数据库控制功能。是用来设置或更改数据库用户或角色权限的语句，包括（grant,deny,revoke等）语句。在默认状态下，只有 sysadmin,dbcreator,db_owner或db_securityadmin等人员才有权力执行DCL
```


#### 3. online DDL
##### 1. 在mysql5.5之前,要修改表结构
1. 创建一张新的临时表，通过alter table 修改表结构
2. 把原表的数据导入新表
3. 删除原表
4. 新表重命名为原表

如果表是大表，这非常繁琐和耗时,而且如果有大龄事务需要访问正在被修改的表，这时候数据库不可用。

##### 2. fast index creation (FIC)
对于辅助索引的创建，InnoDB对表添加一个S锁，这在索引创建过程中不需要重建表，速度提升了。但是**只允许读操作，但是不能写**。也只能用在添加辅助索引。

#### 3. online schema change（OSC）
OSC是facebook开发的一个在线ddl工具，使用php语言。OSC使用了触发器和零时表的方法，具体过程如下:
![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200620osc.jpg)

#### 4. online ddl; 
mysql 5.6提供了online ddl，允许了再ddl的同时，还允许insert update delete等dml操作。
下面几类操作都可以使用 online ddl:    
1. 辅助索引的创建于删除
2. 改变自增长值
3. 添加或者删除外键约束
4. 列的重命名

online ddl 可以有2个参数:
alter table xxx add xxxIndexName (col1,col2,...) 
algorithm={default|implace|copy}
lock={default|NONE|shared|exclusive}


##### 1. algorithm:
1. copy表示使用创建零时表的方式，
2. inplace表示索引的删除和创建不需要添加零时表。

##### 2. lock:
1. none 
执行创建删除索引时候，不添加任何锁，事务的读写操作都不会阻塞，并发度最高

2. share
执行创建删除索引时候，对表加S锁，读事务不阻塞，写事务会等待

3. exclusive
加s锁，读写都不能进行

4. default
default模式会自动判断用哪个上面的模式，先判断是否为none，如果不行，能不能为share，然后尝试exclusive，有限选择最大的并发度。

##### 3. online ddl 原理

在执行online ddl的时候，将 insert update delete等操作日志写入一个日志中，创建完索引后再将日志重做到表上。日志大小由innodb_online_alter_log_max_size控制。如果日志满了 也会报错。

```sql
mysql> show variables like "%alter_log_max%%";
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| innodb_online_alter_log_max_size | 134217728 |
+----------------------------------+-----------+
1 row in set (0.00 sec)


```



#### 3. cardinality 帮助建立索引
> cardinality /kɑːdɪ'nælɪtɪ/ n.基数
cardinality表示去重后的个数，建立索引的前提是数据是**高选择性的**。 举一个反例，性别就不是高选择性的。像名字重复就少很多，建立索引就还不错。   

**cardinality/总行数** 可以用来判断选择性高不高。

cardinality不是精确值只是一个估计值，表中1/16的数据变化后，或者内部计数器stat_modified_counter大于2000000000就更新这个值。 手动更新可以使用 analyze table 命令


#### 4. 联合索引
```sql
mysql> mysql> show create table t;
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                |
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `a` int(11) NOT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`),
  KEY `idx_ab` (`a`,`b`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```


上面 表t建立了一个联合索引idx_ab.

**B+树索引大概是下面这个样子**
![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200625-unitedIndex.jpg)

索引数据按照(a,b)进行**顺序**存放 (1,1),(1,2),(2,1),(2,4),(3,1),(3,2),可以看到 如果a的值一定，比如 (1,1),(1,2),b值是顺序存放的，但是整体来看b值不是顺序存放的: 1,2,1,4,1,2.

所以在我们在使用 **select * from t where a = 1 and b = 2** 的时候是可以用到索引的，但是 **select * from t where  b = 2** 就用不到索引了。 这就是我们常说的索引只能使用**左前缀**。

**联合索引对排序也有帮助**
```sql
mysql> show create table t1;
mysql> show create table t1;
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                        |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t1    | CREATE TABLE `t1` (
  `user_id` int(11) DEFAULT NULL,
  `buy_date` date DEFAULT NULL,
  KEY `idx_u_date` (`user_id`,`buy_date`),
  KEY `user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from t1;
+---------+------------+
| user_id | buy_date   |
+---------+------------+
|       1 | 2020-01-01 |
|       1 | 2020-01-04 |
|       1 | 2020-01-05 |
|       2 | 2020-01-02 |
|       2 | 2020-01-05 |
+---------+------------+
5 rows in set (0.01 sec)

```
例如上面 buy_date 是购买日期.

下面第一条sql只使用索引user_id的时候会有 mysql会执行一次排序(filesort), 第二条sql使用联合索引idx_u_date，就没有filesort，**因为idx_u_date联合索引已经将date进行了排序**。

```sql
mysql> explain select * from t1 force index(user_id) where user_id = 1 order by buy_date desc;
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | user_id       | user_id | 5       | const |    3 |   100.00 | Using index condition; Using filesort |
+----+-------------+-------+------------+------+---------------+---------+---------+-------+------+----------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from t1 force index(idx_u_date) where user_id = 1 order by buy_date desc;
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | idx_u_date    | idx_u_date | 5       | const |    3 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)

```


#### 5. 覆盖索引(covering index)
覆盖索引就是从辅助索引仲就可以得到需要查询的数据，不需要再查一遍聚集索引，还有聚集索引不包含一行记录的所有信息，只包含了很少的信息，所以比聚集索引小很多，所有覆盖索引会减少很多的io操作。

**extra: Using index** 就表示使用到了覆盖索引。
```sql
mysql> explain select * from t1 force index(idx_u_date) where user_id = 1 order by buy_date desc;
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
| id | select_type | table | partitions | type | possible_keys | key        | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | t1    | NULL       | ref  | idx_u_date    | idx_u_date | 5       | const |    3 |   100.00 | Using where; Using index |
+----+-------------+-------+------------+------+---------------+------------+---------+-------+------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)


```


# 锁
>锁机制是用来管理对共享资源的并发访问
**innodb提供了一致性的非锁定读，行锁，可以同时得到并发性和一致性。**

## 1. 锁的类型:

### 1. 共享锁，排它锁
Innodb实现了两种标准的行级锁：   
1. 共享锁(S Lock): 允许事务读取一行    
2. 和排它锁(X Lock)：允许事务删除或者更新一行  


下面是两种锁的兼容性:
**注意x锁和s锁都是行锁，兼容是指对同一行(row)记录的兼容情况**
![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200624-sxlock.jpg)



### 2. 意向锁
意向锁是将要锁定的对象分为很多层次，对某一个细粒度的对象上锁，需要对粗粒度的对象上锁，如果将数据库 分为三层： **表-页-记录**的话，那么要对某个记录r上X锁，就需要分别对**表和页**上加IX锁，然后再对r加X锁，如果任何一部分导致等待，改操作需要等待粗粒度的锁的完成。 例如： 如果**事务1**对表加了S锁，表上已经存在S锁了，如果之后**事务2**要对记录r加上加X锁，就需要先对表加IX锁，但是IX锁与S锁不兼容，所以需要等待表S锁释放

IX IS S X 互斥性    
![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200624-ixislock.jpg)