---
layout: post
title: mysql master-slave
date: 2019-11-16 14:32:00
categories:  search java
---

在master上
```sql
create user 'replication'@'%' identified by 'replication1234';
grant replication slave on *.* to 'replication'@'%';
flush privileges; 
```


在slave上
```sql
CHANGE MASTER TO MASTER_HOST='localhost', MASTER_USER='replication', 
MASTER_PORT='4106',
MASTER_PASSWORD='replication1234', MASTER_LOG_FILE='ON.000002', MASTER_LOG_POS=357;
```