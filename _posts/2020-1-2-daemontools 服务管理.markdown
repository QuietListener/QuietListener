---
layout: post
title: daemontools 服务管理
date: 2019-12-24 14:32:00
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

 ## 5.配置开机自启动
 1. 安装 csh 

 ```shell
 yum install csh
 ```

 2. 修改/etc/rc.local
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

3. 启动svscanboot
```shell
[root@iZb daemontools]# bash -x /etc/rc.local 
+ touch /var/lock/subsys/local
+ csh -cf '/command/svscanboot &'

[root@iZbp15pf7sr2cgwbyqi1inZ daemontools]# ps -ef | grep svs
root      8871     1  0 14:51 pts/3    00:00:00 /bin/sh /command/svscanboot
root      8873  8871  0 14:51 pts/3    00:00:00 svscan /service
root      9024  6265  0 14:51 pts/3    00:00:00 grep svs

```

可以看到 /command/svscanboot的pid为8871，**svscan /service**pid为8873他的父pid是8871,
svscan作为svscanboot的子进程在运行。





