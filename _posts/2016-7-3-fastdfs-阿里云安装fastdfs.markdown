---
layout: post
title:  fastdfs-阿里云安装fastdfs
date:   2016-7-3 11:34:00
categories: 数据库 
---
公司现在的文件备份方式是2台机器，使用rsync进行同步，扩展性和容错性都不能满足需求。
调研了一下 fastdfs，能满足公司也无需求。

### 系统环境
```
[www@iZ237c9otg7Z programs]$ uname -a
Linux iZ237c9otg7Z 2.6.32-504.8.1.el6.x86_64 #1 SMP Wed Jan 28 21:11:36 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

## 安装
### 安装libfastcommon

```
[www@iZ237c9otg7Z programs]$ git clone https://github.com/happyfish100/libfastcommon.git
[www@iZ237c9otg7Z libfastcommon]$ ./make.sh 
[root@iZ237c9otg7Z libfastcommon]# sudo ./make.sh install
```

###安装fastdfs

```
[www@iZ237c9otg7Z programs]$ git clone https://github.com/happyfish100/fastdfs.git
[www@iZ237c9otg7Z fastdfs]$ ./make.sh 
[www@iZ237c9otg7Z fastdfs]$ ./make.sh  install
```


##配置并启动 tracker服务和storage服务
fastdfs源码在/home/www/programs/fastdfs
配置文件在/home/www/programs/fastdfs/conf

```
[www@iZ237c9otg7Z fastdfs]$ pwd
/home/www/programs/fastdfs
[www@iZ237c9otg7Z fastdfs]$ cd conf/
[www@iZ237c9otg7Z conf]$ pwd
/home/www/programs/fastdfs/conf
[www@iZ237c9otg7Z conf]$ ls
anti-steal.jpg  client.conf  http.conf  mime.types storage.conf  storage_ids.conf  tracker.conf

```
####编辑tracker配置文件storage.conf

``` 
port=22122 # the tracker server port
base_path=/var/www/data/fdfs #the base path to store data and log files
```

#### 启动tracker
启动后的情况可以看log,log在base_path(/var/www/data/fdfs/logs/trackerd.log)中

```
[www@iZ237c9otg7Z conf]$ ../restart.sh /usr/bin/fdfs_trackerd /home/www/programs/fastdfs/conf/tracker.conf 
```


#### 编辑存储节点配置文件storage.conf

```
tracker_server=10.161.182.210:22122
port=23001  # the storage server port
base_path=/var/www/data/fdfs_store1  # the base path to store data and log files
store_path0=/var/www/data/fdfs_store1 # if store_path0 not exists, it's value is base_path,the paths must be exist
```
####启动存储节点
启动后的情况也是看log,log在base_path(/var/www/data/fdfs_store1/logs/trackerd.log)中


```
[www@iZ237c9otg7Z conf]$ pwd
/home/www/programs/fastdfs/conf
[www@iZ237c9otg7Z conf]$ ../restart.sh /usr/bin/fdfs_storaged /home/www/programs/fastdfs/conf/storage.conf 
```

##测试

####配置client.conf

```
base_path=/var/www/data/fdfs_client # the base path to store log files
tracker_server=127.0.0.1:22122
```
###使用fdfs_test测试上传一个问题 anti-steal.jpg  

```

www@iZ237c9otg7Z conf]$ fdfs_test ~/programs/fastdfs/conf/client.conf upload anti-steal.jpg   
This is FastDFS client test program v5.08

Copyright (C) 2008, Happy Fish / YuQing

FastDFS may be copied only under the terms of the GNU General
Public License V3, which may be found in the FastDFS source kit.
Please visit the FastDFS Home Page http://www.csource.org/ 
for more detail.

[2016-07-04 17:09:39] DEBUG - base_path=/var/www/data/fdfs_client, connect_timeout=30, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

tracker_query_storage_store_list_without_group: 
	server 1. group_name=, ip_addr=10.161.182.210, port=23001

group_name=group1, ip_addr=10.161.182.210, port=23001
storage_upload_by_filename
group_name=group1, remote_filename=M00/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439.jpg
source ip address: 10.161.182.210
file timestamp=2016-07-04 17:09:39
file size=23981
file crc32=2553063749
example file url: http://10.161.182.210/group1/M00/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439.jpg
storage_upload_slave_by_filename
group_name=group1, remote_filename=M00/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439_big.jpg
source ip address: 10.161.182.210
file timestamp=2016-07-04 17:09:39
file size=23981
file crc32=2553063749
example file url: http://10.161.182.210/group1/M00/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439_big.jpg
```

###在本地访问上传到文件
在输出信息中可以看到 上传文件被重命名了，remote_filename=M00/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439_big.jpg
可以在本地硬盘中看到当前上传的文件

```
[www@iZ237c9otg7Z conf]$ ls /var/www/data/fdfs_store1/data/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439_big.jpg -lah
-rw-r--r-- 1 www www 24K 7月   4 17:09 /var/www/data/fdfs_store1/data/00/00/CqG20ld6J9OAOEG7AABdrZgsqUU439_big.jpg
[www@iZ237c9otg7Z conf]$ 

```



<div class="ds-thread" data-thread-key="11" data-title="moosefs-阿里云安装moosefs" data-url="https://quietlistener.github.io/ruby/2016/07/03/rmoosefs-阿里云安装moosefs.html"></div>
       
<script type="text/javascript">
        var duoshuoQuery = {short_name:"quietlistener"};
	(function() {
		var ds = document.createElement('script');
		ds.type = 'text/javascript';ds.async = true;
		ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
		ds.charset = 'UTF-8';
		(document.getElementsByTagName('head')[0] 
		 || document.getElementsByTagName('body')[0]).appendChild(ds);
	})();
</script>
