---
layout: post
title:  centos安装mysql5.7和java
date:   2018-5-19 14:32:00
categories: 幂等 mysql
---

### 1. centos 免安装mysql5.7
#### 1. 下载:

 > wget --no-check-certificate https://dev.mysql.com//Downloads/MySQL-5.7/mysql-5.7.21-linux-glibc2.12-x86_64.tar.gz

#### 2. 安装
1. 安装依赖库 yum install libaio*
2. mysql-5.7.21-linux-glibc2.12-x86_64.tar 解压放到 /opt/mysql-5.7.21/
3. 初始化，记住localhost后的初始密码

```
[root@izm5e4x1zl97yhhpepmsv4z mysql-5.7.21]# bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql/mysql-5.7.21  --datadir=/usr/local/mysql/data
2018-01-23T03:02:16.329446Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2018-01-23T03:02:18.383260Z 0 [Warning] InnoDB: New log files created, LSN=45790
2018-01-23T03:02:18.614416Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2018-01-23T03:02:18.727035Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: d2ce084d-ffe9-11e7-941e-00163e049d34.
2018-01-23T03:02:18.729139Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2018-01-23T03:02:18.732182Z 1 [Note] A temporary password is generated for root@localhost: tyjsc?34kcqT
```


4. Mysql5.7默认使用ssl安全连接 执行下面语句产生key
```
[root@izm5e4x1zl97yhhpepmsv4z mysql-5.7.21]# ./bin/mysql_ssl_rsa_setup  --datadir=/usr/local/mysql/data
Generating a 2048 bit RSA private key
......................+++
...........+++
writing new private key to 'ca-key.pem'
-----
Generating a 2048 bit RSA private key
.......................................................+++
.........................................................+++
writing new private key to 'server-key.pem'
-----
Generating a 2048 bit RSA private key
.....................+++
.....+++
writing new private key to 'client-key.pem'
-----

```
5. 创建 配置文件
```
[root@izm5e4x1zl97yhhpepmsv4z support-files]# pwd
/usr/local/mysql/mysql-5.7.21/support-files
[root@izm5e4x1zl97yhhpepmsv4z support-files]# touch my-default.cnf
[root@izm5e4x1zl97yhhpepmsv4z support-files]# vim my-default.cnf 
my_default.cnf 内容如下

[mysqld]
user = mysql
port = 3306
server_id = 1
socket=/tmp/mysql.sock
basedir =/usr/local/mysql/mysql-5.7.21
datadir =/usr/local/mysql/data
character-set-server=utf8

[client]
socket=/tmp/mysql.sock
```

6. 拷贝一些文件
```
[root@izm5e4x1zl97yhhpepmsv4z support-files]# cp my-default.cnf /etc/my.cnf
[root@izm5e4x1zl97yhhpepmsv4z support-files]# cp mysql.server /etc/init.d/mysqld
[root@izm5e4x1zl97yhhpepmsv4z support-files]# chmod 755 /etc/init.d/mysqld
```

7. 编辑 /etc/init.d/mysqld配置
```
basedir=/usr/local/mysql/mysql-5.7.21/
datadir=/usr/local/mysql/data/
```

8. 启动mysqld 
```
[root@izm5e4x1zl97yhhpepmsv4z support-files]# service mysqld start
Starting MySQL.Logging to '/usr/local/mysql/data/izm5e4x1zl97yhhpepmsv4z.err'.
                                                          [  OK  ]
 ``` 
 也可以用下面方式启动 
*  /etc/init.d/mysqld start    
*  nohup ./bin/mysqld --defaults-file=./support-files/my-default.cnf &


**9. Mysql客户端登录**
**建立软连接方便使用**

```
[root@izm5e4x1zl97yhhpepmsv4z support-files]# ln -s /usr/local/mysql/mysql-5.7.21/bin/mysql /usr/bin 
```

登录初始密码是 "bin/mysqld --initialize" 时候生成的随机密码 **tyjsc?34kcqT**
**登录并重新设置密码(必须重新设置root密码)**

```
[root@izm5e4x1zl97yhhpepmsv4z support-files]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.21

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
//重新设置密码
mysql> set password=password('root');
Query OK, 0 rows affected, 1 warning (0.00 sec)
//设置权限
mysql> grant all privileges on *.* to 'root'@'%' identified by 'root'
            with grant option ;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql>
```



### 2. 免安装java8
1. 解压jdk-8u162-linux-x64.tar.gz 到 /home/www/programs/jdk1.8.0_162

2. 在 ~/.bash_profile中设置 JAVA_HOME 环境变量
JAVA_HOME=/home/www/programs/jdk1.8.0_162
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$JAVA_HOME/bin

3. source ~/.bash_profile


