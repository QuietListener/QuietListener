---
layout: post
title: daemontools 服务管理
date: 2020-1-2 14:32:00
categories:  linux 服务管理
---

# why  
1.有的服务是非常关键的，比如zookeeper，mysql等，当down掉后必须重启。daemontools是一个工具集，当服务kill掉或者down掉后可以自动重启。

# 安装
## 1. 下载 解压
wget http://cr.yp.to/daemontools/daemontools-0.76.tar.gz    
tar -zxvf daemontools-0.76.tar.gz   

## 2. 修改一下文件防止编译安装时候的错误
vim src/conf-cc    
在最后一行加上  **-include /usr/include/errno.h**    
最后的src/conf-cc 如下所示: 

```shell
gcc -O2 -Wimplicit -Wunused -Wcomment -Wchar-subscripts -Wuninitialized -Wshadow -Wcast-qual -Wcast-align -Wwrite-strings -include /usr/include/errno.h
```
## 3. 编译安装
**需要切换到root用户**
[root@iZbZ daemontools]# bash package/install
如果没有问题就装好了

## 4. 检查一下安装结果
安装成功后，会在根目录下创建两个目录，/command存放命令,/service是一个空目录
```shell
[root@iZbZ daemontools]# ls -ld  /service/ /command/
drwxr-xr-x 2 root root 4096 Jan  2 14:36 /command/
drwxr-xr-x 2 root root 4096 Jan  2 13:47 /service/
```

```shell
[root@iZbZ daemontools]# ls -l  /command/
total 0
lrwxrwxrwx 1 root root 51 Jan  2 14:36 envdir -> /home/www/programs/admin/daemontools/command/envdir
lrwxrwxrwx 1 root root 54 Jan  2 14:36 envuidgid -> /home/www/programs/admin/daemontools/command/envuidgid
lrwxrwxrwx 1 root root 51 Jan  2 14:36 fghack -> /home/www/programs/admin/daemontools/command/fghack
lrwxrwxrwx 1 root root 53 Jan  2 14:36 multilog -> /home/www/programs/admin/daemontools/command/multilog
lrwxrwxrwx 1 root root 53 Jan  2 14:36 pgrphack -> /home/www/programs/admin/daemontools/command/pgrphack
lrwxrwxrwx 1 root root 58 Jan  2 14:36 readproctitle -> /home/www/programs/admin/daemontools/command/readproctitle
lrwxrwxrwx 1 root root 52 Jan  2 14:36 setlock -> /home/www/programs/admin/daemontools/command/setlock
lrwxrwxrwx 1 root root 54 Jan  2 14:36 setuidgid -> /home/www/programs/admin/daemontools/command/setuidgid
lrwxrwxrwx 1 root root 54 Jan  2 14:36 softlimit -> /home/www/programs/admin/daemontools/command/softlimit
lrwxrwxrwx 1 root root 54 Jan  2 14:36 supervise -> /home/www/programs/admin/daemontools/command/supervise
lrwxrwxrwx 1 root root 48 Jan  2 14:36 svc -> /home/www/programs/admin/daemontools/command/svc
lrwxrwxrwx 1 root root 49 Jan  2 14:36 svok -> /home/www/programs/admin/daemontools/command/svok
lrwxrwxrwx 1 root root 51 Jan  2 14:36 svscan -> /home/www/programs/admin/daemontools/command/svscan
lrwxrwxrwx 1 root root 55 Jan  2 14:36 svscanboot -> /home/www/programs/admin/daemontools/command/svscanboot
lrwxrwxrwx 1 root root 51 Jan  2 14:36 svstat -> /home/www/programs/admin/daemontools/command/svstat
lrwxrwxrwx 1 root root 51 Jan  2 14:36 tai64n -> /home/www/programs/admin/daemontools/command/tai64n
lrwxrwxrwx 1 root root 56 Jan  2 14:36 tai64nlocal -> /home/www/programs/admin/daemontools/command/tai64nlocal
```   

##  5.配置开机自启动
1. 修改/etc/rc.local   
 在 rc.local 中加入一行 **nohup bash /command/svscanboot &** 可以开机自启动。

```shell
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
nohup bash /command/svscanboot &
```

2. 启动svscanboot   

```shell
[root@iZb daemontools]# bash -x /etc/rc.local 
+ touch /var/lock/subsys/local
+ csh -cf '/command/svscanboot &'

[root@iZbp15pf7sr2cgwbyqi1inZ daemontools]# ps -ef | grep svs
root      8871     1  0 14:51 pts/3    00:00:00 /bin/sh /command/svscanboot
root      8873  8871  0 14:51 pts/3    00:00:00 svscan /service
root      9024  6265  0 14:51 pts/3    00:00:00 grep svs

```

可以看到 /command/svscanboot的pid为8871，**svscan** 的pid为8873他的父pid是8871,
svscan作为svscanboot的子进程在运行。


# how(怎么工作)
## 三个命令
主要有三个程序需要了解
### 1. svscanboot   
一般是在rc.local中配置，开机启动 svscanboot会启动svscan子进程
### 2. svscan  
svscan会去监听/service目录并启动为每一服务启动supervise进程
### 3. supervise  
supervise会去执行/service目录下的服务的run脚本，如果服务down掉了，会每隔几秒重启一下服务。

## 怎么将服务交给supervise监控
### hello项目
假如我们有一个项目叫hello放在目录/home/www/projects/test/hello
我们将启动的代码放在了 run里面
```shell
[root@iZbp15pf7sr2cgwbyqi1inZ hello]# pwd
/home/www/projects/test/hello
[root@iZbp15pf7sr2cgwbyqi1inZ hello]# ls -lhs 
total 144K
136K -rw-r--r-- 1 root root 132K Jan  2 15:06 h.log
4.0K -rwxr-xr-x 1 www  www    63 Jan  2 13:40 run
```

**run的内容**
run就是打印一句hello 就推出。
```shell
#!/bin/bash
echo "hello `date +%Y_%m_%d%t%H:%M:%S` " >> h.log
```
### 将hello项目交给supervise来管理
如果我们要将 hello 这个项目交给supervise，我们预期是: **每个几秒就会启动一次run，我们会在h.log中看到打印结果。**

#### 1. 怎么做?

我们只需将 /home/www/projects/test/hello/ 软连接到 /service/hello
> ln -sv /home/www/projects/test/hello/ /service/hello
svscan会扫描/service/目录，当发现目录有变化，svscan会启动给一个supervise来执行 run脚本

#### 2. 结果
我们看到 启动了一个supervise来 执行 hello项目的run脚本
```shell
[root@iZbp hello]# ps -ef | grep hello
root      9837  9832  0 14:56 pts/3    00:00:00 supervise hello
root     13669  6265  0 15:13 pts/3    00:00:00 grep hello
```
**看一下log** 与我们预期相符。
```shell
[root@iZbp15pf7sr2cgwbyqi1inZ hello]# tail h.log 
hello 2020_01_02	15:14:44 
hello 2020_01_02	15:14:45 
hello 2020_01_02	15:14:46 
hello 2020_01_02	15:14:47 
hello 2020_01_02	15:14:48 
hello 2020_01_02	15:14:49 
hello 2020_01_02	15:14:50 
hello 2020_01_02	15:14:51 
hello 2020_01_02	15:14:52 
hello 2020_01_02	15:14:53 

```

supervise生成了一个**supervise** 目录 ，里面存放的进程的状态信息。
```shell
[root@iZbp hello]# ls -lhs 
total 160K
152K -rw-r--r-- 1 root root 147K Jan  2 15:16 h.log
4.0K -rwxr-xr-x 1 www  www    63 Jan  2 13:40 run
4.0K drwx------ 2 root root 4.0K Jan  2 15:16 supervise
[root@iZbp hello]# ls -lah supervise/
total 12K
drwx------ 2 root root 4.0K Jan  2 15:16 .
drwxrwxr-x 3 www  www  4.0K Jan  2 15:07 ..
prw------- 1 root root    0 Jan  2 14:56 control
-rw------- 1 root root    0 Jan  2 13:38 lock
prw------- 1 root root    0 Jan  2 13:38 ok
-rw-r--r-- 1 root root   18 Jan  2 15:16 status

```

## 让服务run在指定用户权限下
supervise都是使用root来run的，有时候我们的程序需要跑在指定的user权限下。
我们怎么做，

### 例如mysql跑在www下
使用 exec 先切到www下，再运行服务。   
```shell
#!/bin/bash
echo "restart mysqld at  `date +%Y_%m_%d%t%H:%M:%S` " >>  run.log
base=/home/www/programs/mysql-5.7.21-linux-glibc2.12-x86_64
exec su - www -c "exec ${base}/bin/mysqld  --defaults-file=${base}/support-files/my_default.cnf >> run.log"
```

# 几个有用的命令
  daemontools 有几个有用的命令。

## 1. svstat查看状态   

```shell
[root@iZbp15pf7sr2cgwbyqi1inZ ~]# svstat  /service/mysqld
/service/mysqld: up (pid 9839) 1668 seconds
```

## 2.  svc -d /service/serviceName 停止监控服务   

```shell
[root@iZbp15pf7sr2cgwbyqi1inZ ~]# svc -d   /service/mysqld
[root@iZbp15pf7sr2cgwbyqi1inZ ~]# svstat  /service/mysqld
/service/mysqld: down 4 seconds, normally up
```
当mysql服务停止后不会重启mysqld服务。


## 3.  svc -u /service/serviceName 开启监控服务   

```shell
[root@iZbp15pf7sr2cgwbyqi1inZ ~]# svc -u  /service/mysqld
[root@iZbp15pf7sr2cgwbyqi1inZ ~]# svstat  /service/mysqld
/service/mysqld: up (pid 15404) 2 seconds
```
当mysql服务停止后重启mysqld服务。


## 4.彻底删除某个服务
将/service目录下对应的软连接删除即可

