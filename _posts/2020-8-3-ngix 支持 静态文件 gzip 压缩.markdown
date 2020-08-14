---
layout: post
title: 2020-8-3-ngix 支持 静态文件 gzip 压缩
date: 2020-5-8 14:32:00
categories:  java maven
---
```shell
gzip on;
# 值为1.0和1.1 代表是否压缩http协议1.0，选择1.0则1.0和1.1都可以压缩 
gzip_http_version   1.1;

#分配内存的规则，每次分配4个16K的内存来存放gzip
gzip_buffers 4 16K;

#level越大压缩率越高，越耗cpu
gzip_comp_level 4;

#2k以上的才压缩，设置得太小没有意义，有可能压缩了反而更大
gzip_min_length 2k; 

#客户端与服务器协商 根据头信息来判断是不是需要压缩，因为有的客户端器可能不支持gzip压缩
gzip_vary on;

# 可以将文件与压缩为.gz文件,nginx直接读取.gz, 文件不用每次都要重新压缩 
# 压缩命令如下: gzip -c -9 jquery-1.11.2.min.js > jquery-1.11.2.min.js.gz
# 需要安装http_gzip_static_module插件 ./configure --prefix=/usr/local/nginx1  --with-http_ssl_module --with-http_gzip_static_module 
gzip_static on;  

#charset koi8-r;

#这个参数用于指定当 http请求来自代理服务器时（如何判断？请求头里包含 VIA 这个参数就认为这个请求来自代理服务器）基于代理服务器的类型来决定是否进行压缩。
gzip_proxied any;

#哪些类型的需要压缩
gzip_types text/plain application/javascript application/css text/css application/xml text/javascript application/x-httpd-php

```