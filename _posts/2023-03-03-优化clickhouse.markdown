---
layout: post
title: 优化clickhouse
date: 2023-03-03 14:32:00
categories:  开发
---

# 问题
线上有台clickhouse，突然cpu经常100% 上去看了一下。 所有查询都是全表扫描。 

看了一下 primary key的设计 是按server_time 来的，但是查询中没有一个是与server_time相关了。所以导致了全局扫描。

```sql

CREATE TABLE table_xxx
(
    `bcz_uid` UInt32,
    `user_id` UInt32,
    `sku_id` UInt32,
    `term_id` UInt32,
    `chapter_id` UInt32,
    `lesson_id` UInt32,
    `page_id` UInt32,
    `event_time` DateTime,
    `time_type` UInt16,
    `time` DateTime,
    `extra` String,
    `server_time` DateTime,
    `created_at` DateTime,
    `updated_at` DateTime,
    PROJECTION prj_term_id_bcz_uid_event_time
    (
        SELECT *
        ORDER BY (term_id, bcz_uid, event_time)
    )
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(server_time)
ORDER BY event_time
SETTINGS index_granularity = 8192

```


而查询语句多是这样的
```sql
SELECT
  bcz_uid,
  user_id,
  sku_id,
  term_id,
  chapter_id,
  lesson_id,
  page_id,
  event_time,
  server_time,
  time_type,
  time,
  extra,
  updated_at,
  created_at
FROM
  table_xxx
WHERE
  (
    bcz_uid = 8061112411
    AND time_type IN (200112, 200422)
    AND lesson_id IN (28118, 292221, 28227, 2222, 2862, 1289)
    AND term_id = 871
    AND sku_id = 37
  ) FORMAT TabSeparatedWithNamesAndTypes
```


### 优化之前执行 如下sql
```sql

SELECT
                                                   bcz_uid,
                                                   user_id,
                                                   sku_id,
                                                   term_id,
                                                   chapter_id,
                                                   lesson_id,
                                                   page_id,
                                                   event_time,
                                                   server_time,
                                                   time_type,
                                                   time,
                                                   extra,
                                                   updated_at,
                                                   created_at
                                               FROM table_xxx
                                               WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149,151))


...省略
...省略
 Showed first 10000.

100912 rows in set. Elapsed: 3.137 sec. Processed 230.84 million rows, 13.55 GB (73.59 million rows/s., 4.32 GB/s.)
```
可以看到 结果是全表扫描, "processed 230.84 million row" ,2亿多行。


### 执行explain看一下情况

```sql
EXPLAIN indexes = 1
SELECT
    bcz_uid,
    user_id,
    sku_id,
    term_id,
    chapter_id,
    lesson_id,
    page_id,
    event_time,
    server_time,
    time_type,
    time,
    extra,
    updated_at,
    created_at
FROM course_event_lesson_study
WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149, 151))

Query id: 4c2026e6-9227-473c-b5ff-b53dbd7c587d

┌─explain─────────────────────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY))                                 │
│   Filter (WHERE)                                                            │
│     SettingQuotaAndLimits (Set limits and quota after reading from storage) │
│       ReadFromMergeTree                                                     │
│       Indexes:                                                              │
│         MinMax                                                              │
│           Condition: true                                                   │
│           Parts: 6/6                                                        │
│           Granules: 28191/28191                                             │
│         Partition                                                           │
│           Condition: true                                                   │
│           Parts: 6/6                                                        │
│           Granules: 28191/28191                                             │
│         PrimaryKey                                                          │
│           Condition: true                                                   │
│           Parts: 6/6                                                        │
│           Granules: 28191/28191                                             │
└─────────────────────────────────────────────────────────────────────────────┘

17 rows in set. Elapsed: 0.009 sec. 

c7350838e38d :) 

```
可以发现 Granules: 28191/28191 表示使用了所有的Granules(颗粒，每个Granules包含了8192行，在创建表的时候指定的)。


## 修改优化方案
### 可选优化方案 官方的优化方案有3个
#### 1. 新建表
1. 新建一个表 table_ooo 使用不同主键。
2. 使用insert into table ooo select * from table_xxx 将数据导入新表。
查询的时候需要使用新表。

**创建新表 并导入数据**
```sql
CREATE TABLE table_ooo
(
    `bcz_uid` UInt32,
    `user_id` UInt32,
    `sku_id` UInt32,
    `term_id` UInt32,
    `chapter_id` UInt32,
    `lesson_id` UInt32,
    `page_id` UInt32,
    `event_time` DateTime,
    `time_type` UInt16,
    `time` DateTime,
    `extra` String,
    `server_time` DateTime,
    `created_at` DateTime,
    `updated_at` DateTime
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(server_time)
PRIMARY KEY (term_id, bcz_uid)
ORDER BY (term_id, bcz_uid, server_time)
SETTINGS index_granularity = 8192

Query id: 959d00e5-8784-4f25-982d-511f9e01a092

Ok.

0 rows in set. Elapsed: 0.071 sec. 

c7350838e38d :) insert into table_ooo select * from course_event_lesson_study;

INSERT INTO table_ooo SELECT *
FROM course_event_lesson_study

Query id: be251f00-964b-45ba-8643-2b822053449c

Ok.

0 rows in set. Elapsed: 199.634 sec. Processed 230.84 million rows, 14.46 GB (1.16 million rows/s., 72.45 MB/s.)
```

现在同样的sql在新表执行一下
```

 SELECT
                                                                   bcz_uid,
                                                                   user_id,
                                                                   sku_id,
                                                                   term_id,
                                                                   chapter_id,
                                                                   lesson_id,
                                                                   page_id,
                                                                   event_time,
                                                                   server_time,
                                                                   time_type,
                                                                   time,
                                                                   extra,
                                                                   updated_at,
                                                                   created_at
                                                               FROM table_ooo
                                                               WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149,151))

......省略
......省略
 Showed first 10000.

100912 rows in set. Elapsed: 0.438 sec. Processed 621.19 thousand rows, 12.57 MB (1.42 million rows/s., 28.69 MB/s.)
```

这一次只加载了  60多万行。 Processed 621.19 thousand rows。 比以前快了很多。

**explain 看看效果**

```sql
EXPLAIN indexes = 1
SELECT
    bcz_uid,
    user_id,
    sku_id,
    term_id,
    chapter_id,
    lesson_id,
    page_id,
    event_time,
    server_time,
    time_type,
    time,
    extra,
    updated_at,
    created_at
FROM table_ooo
WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149, 151))

Query id: 5acfd18b-9988-48e4-9f68-e02e501a889c

┌─explain────────────────────────────────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY))                                            │
│   Filter (WHERE)                                                                       │
│     SettingQuotaAndLimits (Set limits and quota after reading from storage)            │
│       ReadFromMergeTree                                                                │
│       Indexes:                                                                         │
│         MinMax                                                                         │
│           Condition: true                                                              │
│           Parts: 17/17                                                                 │
│           Granules: 28250/28250                                                        │
│         Partition                                                                      │
│           Condition: true                                                              │
│           Parts: 17/17                                                                 │
│           Granules: 28250/28250                                                        │
│         PrimaryKey                                                                     │
│           Keys:                                                                        │
│             term_id                                                                    │
│             bcz_uid                                                                    │
│           Condition: and((term_id in [606, 606]), (bcz_uid in [140703747, 140703747])) │
│           Parts: 6/17                                                                  │
│           Granules: 76/28250                                                           │
└────────────────────────────────────────────────────────────────────────────────────────┘

20 rows in set. Elapsed: 0.012 sec. 

c7350838e38d :) 

```
可以看到 “76/28250” 28250个 颗粒(Granules)只需要load 76个大大加快了速度。

#### 2.使用 物化视图
物化视图会生成一个隐藏表。原表的数据会与这个隐藏表自动保持同步。

![部署](https://clickhouse.com/docs/assets/images/sparse-primary-indexes-09b-0a3561fdefc32a8186cd253bd350ee1e.png)

**生成物化视图**
```sql
c7350838e38d :) CREATE MATERIALIZED VIEW mv_term_id_user_id
                ENGINE = MergeTree
                PARTITION BY toYYYYMM(server_time)
                PRIMARY KEY (term_id, bcz_uid)
                ORDER BY (term_id, bcz_uid, server_time)
                SETTINGS index_granularity = 8192
                POPULATE AS SELECT * FROM course_event_lesson_study

CREATE MATERIALIZED VIEW mv_term_id_user_id
ENGINE = MergeTree
PARTITION BY toYYYYMM(server_time)
PRIMARY KEY (term_id, bcz_uid)
ORDER BY (term_id, bcz_uid, server_time)
SETTINGS index_granularity = 8192 POPULATE AS
SELECT *
FROM course_event_lesson_study

Query id: 349f0daa-4de4-445f-99b4-ce16db998e1a

Ok.

0 rows in set. Elapsed: 211.889 sec. Processed 230.84 million rows, 14.46 GB (1.09 million rows/s., 68.26 MB/s.)

c7350838e38d :) 


```

下面相通的sql在视图上执行一下。
```
explain indexes=1
SELECT
                    bcz_uid,
                    user_id,
                    sku_id,
                    term_id,
                    chapter_id,
                    lesson_id,
                    page_id,
                    event_time,
                    server_time,
                    time_type,
                    time,
                    extra,
                    updated_at,
                    created_at
                FROM mv_term_id_user_id
                WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149, 151));
...省略
...省略
 Showed first 10000.

100912 rows in set. Elapsed: 0.534 sec. Processed 612.47 thousand rows, 36.16 MB (1.15 million rows/s., 67.72 MB/s.)                
```

可以看到 在物理视图上 很快。explain 再看看
```sql
EXPLAIN indexes = 1
SELECT
    bcz_uid,
    user_id,
    sku_id,
    term_id,
    chapter_id,
    lesson_id,
    page_id,
    event_time,
    server_time,
    time_type,
    time,
    extra,
    updated_at,
    created_at
FROM mv_term_id_user_id
WHERE (bcz_uid = 140703747) AND (term_id = 606) AND (sku_id = 9) AND (time_type = 3001) AND (lesson_id IN (149, 151))

Query id: 465c211a-9e6d-42dc-b213-ea6351db5a82

┌─explain──────────────────────────────────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY))                                              │
│   Filter (WHERE)                                                                         │
│     SettingQuotaAndLimits (Set limits and quota after reading from storage)              │
│       SettingQuotaAndLimits (Lock destination table for MaterializedView)                │
│         ReadFromMergeTree                                                                │
│         Indexes:                                                                         │
│           MinMax                                                                         │
│             Condition: true                                                              │
│             Parts: 10/10                                                                 │
│             Granules: 28254/28254                                                        │
│           Partition                                                                      │
│             Condition: true                                                              │
│             Parts: 10/10                                                                 │
│             Granules: 28254/28254                                                        │
│           PrimaryKey                                                                     │
│             Keys:                                                                        │
│               term_id                                                                    │
│               bcz_uid                                                                    │
│             Condition: and((term_id in [606, 606]), (bcz_uid in [140703747, 140703747])) │
│             Parts: 5/10                                                                  │
│             Granules: 75/28254                                                           │
└──────────────────────────────────────────────────────────────────────────────────────────┘

21 rows in set. Elapsed: 0.010 sec. 

c7350838e38d :) 

```
使用物理视图跳过了很多 Granules ( Granules: 75/28254        )

。






# 参考资料
1. https://clickhouse.com/docs/zh/guides/improving-query-performance/sparse-primary-indexes/
