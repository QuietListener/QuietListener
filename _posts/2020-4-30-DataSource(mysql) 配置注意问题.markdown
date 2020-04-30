---
layout: post
title: DataSource(mysql) 配置注意问题.markdown
date: 2020-4-30 14:32:00
categories:  java  mysql datasource
---

1. 连接池最大连接数
2. 连接池最小连接数
3. 连接connectionTimeout 建立连接时超时时间
4. 连接的idleTimeout 一个连接能在连接池中保存空闲的时间。
5. 连接的maxLifeTime 一个连接能在连接池中保存最大时间。
6. testSql: 坚持一个连接是否是 正常的,通常使用 select 1; 
            这个参数往往和其他参数配合，
            比如 testOnReturn=true 表示在连接返回给连接池时候进行检测 ; testOnBorrow=true 表示在获取连接前进行检测; testWhleIdle=true 表示在连接空闲时候测试。

