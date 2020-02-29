---
layout: post
title: mysql insert ... on duplicate key update .. 不生效(primary key 溢出)
date: 2020-2-25 14:32:00
categories:  java  logback
---

## 1. 背景
 线上一个表，更新一个表的数据使用 **insert ... on duplicate key update ..** 逻辑完全正确不抛异常，
## 2. 找问题
### 1. 怀疑是 事务的原因
开启手动 commit，数据还是有问题。 但是**mysql不报错**

### 2. 尝试直接用insert 插入新数据报primary key 冲突。
```sql
MySQL [xxx]> select  * from `ceshi`  where `customer_id` = 14595639 order by updated_at desc;
+------------+-------------+---------------+--------------+---------------------+---------------------+
| id         | customer_id | word_level_id | last_sync_at | created_at          | updated_at          |
+------------+-------------+---------------+--------------+---------------------+---------------------+
| 1508040152 |    14595639 |             2 |   1582427034 | 2019-10-09 16:57:37 | 2020-02-29 12:12:36 |
| 1513274154 |    14595639 |            31 |   1571132052 | 2019-10-10 14:27:07 | 2019-10-15 17:34:13 |
| 1288154476 |    14595639 |             1 |   1569061257 | 2019-08-20 02:34:32 | 2019-09-21 18:20:57 |
+------------+-------------+---------------+--------------+---------------------+---------------------+
3 rows in set (0.00 sec)

MySQL [xxx]> insert into ceshi values(NULL,14595639,5,1582947260,now(),now()) ;
ERROR 1062 (23000): Duplicate entry '2147483647' for key 'PRIMARY'
MySQL [xxx]> select max(id) from ceshi;
+------------+
| max(id)    |
+------------+
| 2147483647 |
+------------+
1 row in set (0.00 sec)

MySQL [xxx]>
```

**按理说我插入新数据怎么可能primary key 冲突呢?**

表结构

```sql
CREATE TABLE `ceshi` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `customer_id` int(11) NOT NULL,
  `word_level_id` int(11) NOT NULL,
  `last_sync_at` int(11) NOT NULL,
  `created_at` datetime NOT NULL,
  `updated_at` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_customer_id_and_word_level_id` (`customer_id`,`word_level_id`)
) ENGINE=InnoDB AUTO_INCREMENT=2147483647 DEFAULT CHARSET=utf8

```

原来primary key 是int(11) 2147483647 = 2^32-1 溢出了。

## 3. 解决办法
将 id (primary key) 改为 bigint

## 4. 教训
这个表没有分表，已经有109278725条记录，太大了。    
**所以不要使用大表，要将表控制在2000w行以内。**

```sql
MySQL [xxx]> select count(*) from `ceshi` ;
+-----------+
| count(*)  |
+-----------+
| 109278725 |
+-----------+
1 row in set (15.31 sec)

MySQL [xxx]> 

```



