---
layout: post
title: java调用javascript解决一些java难以处理的问题~
date: 2019-12-18 14:32:00
categories:  java javascript
---

# why
java作为静态语言，有的问题处理起来非常痛苦，例如一个非常复杂的并且异构的json字符串中去提取需要的数据就很痛苦。但是javascript天生就是处理json数据能手。这个时候使用java调用js脚本来处理这类数据，js处理完再将数据返回给java，就能发挥他们各自的优势。

# how

