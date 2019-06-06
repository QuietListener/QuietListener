---
layout: post
title:  centos ruby 环境搭建
date:   2016-5-18 16:12:00
categories: ruby 环境
---

1. 安装如下依赖:
yum install -y gcc-c++ patch readline readline-devel zlib zlib-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison iconv-devel

2. 安装rvm
curl -L get.rvm.io | bash -s stable

3. 安装好后执行
source /home/www/.rvm/scripts/rvm

4. 使用rvm安装ruby
 rvm install ruby-2.2.0


