---
layout: post
title:  linux解决so文件找不到的问题
date:   2016-3-1 17:49:00
categories: linux
---
在安装libbpg文件的过程中，需要libpng下载lipng后，make ,make install.然而奇迹还是没有按照我的预期发生
出现错误:**libpng16.so.16.21: cannot open shared object file: No such file or directory**
但是在lib文件夹里找能找到 

```
[root@siskin packing]# ls /usr/local/lib/ -l 
总用量 21936 
-rw-r--r-- 1 root root  1381750 2月  26 17:41 libpng16.a  
-rwxr-xr-x 1 root root      937 2月  26 17:41 libpng16.la  
lrwxrwxrwx 1 root root       19 2月  26 17:41 libpng16.so -> libpng16.so.16.21.0  
lrwxrwxrwx 1 root root       19 2月  26 17:41 libpng16.so.16 -> libpng16.so.16.21.0  
```

**但是程序就是找不到 why?**

linux有一个管理动态库的cache，当程序需要某个动态库的时候就会到这个cache里面去寻找，使用ldconfig命令来创建这个cache
**sudo ldconfig 就能看到所有cache里面的动态库**
```
[root@siskin packing]# ldconfig -v  
ldconfig: /etc/ld.so.conf.d/kernel-2.6.32-431.23.3.el6.x86_64.conf:6: duplicate hwcap 1 nosegneg 
ldconfig: /etc/ld.so.conf.d/kernel-2.6.32-504.16.2.el6.x86_64.conf:6: duplicate hwcap 1 nosegneg 
ldconfig: /etc/ld.so.conf.d/kernel-2.6.32-573.1.1.el6.x86_64.conf:6: duplicate hwcap 1 nosegneg 
ldconfig: 多次给出路径“/usr/lib” 
ldconfig: 多次给出路径“/usr/lib64” 
/usr/lib64/mysql: 
    libmysqlclient.so.16 -> libmysqlclient.so.16.0.0 
    libmysqlclient_r.so.16 -> libmysqlclient_r.so.16.0.0 
```

看到第一行没有: 
>ldconfig: /etc/ld.so.conf.d/kernel-2.6.32-431.23.3.el6.x86_64.conf:6: duplicate hwcap 1 nosegneg 

ldconfig 会到/etc/ld.so.conf.d 下面去找配置文件 我们看看这个目录下面的文件 

```
root@siskin packing]# ls /etc/ld.so.conf.d/ -l 
总用量 24 
-r--r--r--. 1 root root 324 6月  20 2014 kernel-2.6.32-431.20.3.el6.x86_64.conf 
-r--r--r--  1 root root 324 8月   1 2014 kernel-2.6.32-431.23.3.el6.x86_64.conf 
-r--r--r--  1 root root 324 4月  22 2015 kernel-2.6.32-504.16.2.el6.x86_64.conf 
-r--r--r--  1 root root 324 7月  26 2015 kernel-2.6.32-573.1.1.el6.x86_64.conf 
-rw-r--r--  1 root root  17 6月  22 2015 mysql-x86_64.conf 
-rw-r--r--  1 root root  22 9月  24 2011 qt-x86_64.conf 
```

看看这些conf文件的内容,conf文件里面都是库的路径 
``
[root@siskin packing]# cat /etc/ld.so.conf.d/qt-x86_64.conf  
/usr/lib64/qt-3.3/lib 
``

回到我们的问题，我们的libpng16.so.16 在 /usr/local/lib/ 里面 
**我们新建一个 usr-local-lib.conf,并将/usr/local/lib/加入到文件中，执行sudo ldconfig**
**问题解决了，/usr/local/lib/ 中的所有动态库都会加入到系统动态库cache中去**
<hr>


<div class="ds-thread" data-thread-key="287e6d0d98390bd93815852979622c8b" data-title="ldconfig-解决so文件找不到的问" data-url="https://quietlistener.github.io/linux/2016/03/01/ldconfig-%E8%A7%A3%E5%86%B3so%E6%96%87%E4%BB%B6%E6%89%BE%E4%B8%8D%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98.html"></div>
       
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

