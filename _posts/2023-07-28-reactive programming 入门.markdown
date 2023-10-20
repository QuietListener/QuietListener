---
layout: post
title: reactive programming 入门
date: 2023-07-28 14:32:00
categories:  开发
---

# 记录

1. 推和拉的区别
You can also compare the main reactive streams pattern with the familiar Iterator design pattern, as there is a duality to the Iterable-Iterator pair in all of these libraries. One major difference is that, while an Iterator is pull-based, reactive streams are push-based.



# 两种类型的Publisher Flux和Mono
Reactor introduces composable reactive types that implement Publisher but also provide a rich vocabulary of operators: Flux and Mono. A Flux object represents a reactive sequence of 0..N items, while a Mono object represents a single-value-or-empty (0..1) result.

# 参考：


https://projectreactor.io/docs/core/release/reference/