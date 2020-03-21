---
layout: post
title: 吴恩达机器学习记录6 逻辑回归(logistic regression)
date: 2020-3-18 14:32:00
categories:  机器学习
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 1. Logistic Regression 模型
## 1. Sigmoid 函数(也叫 Logistic 函数) 
$$g(z) = \frac1 { 1+e^{-z}} $$

<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200318-logistics-regression1.jpg" width="300"> 

## 2. Logistic Regression 模型

1. hypothesis:
   $ h_θ(x) =  g(θ^T x) = \frac1 { 1+e^{-θ^T x}}$
2. $ 0< h_θ(x)<1 $ 

3. 输出 $h_θ(x)$的意义
以 肿瘤的例子来讲 输入是**肿瘤的大小**,如果一个肿瘤大小是 10，输出 **$h_θ(x)$=0.7**那么表有0.7的概率是良性肿瘤。   
其实就是在输入肿瘤大小为10的情况下 是良性肿瘤的概率 $h_θ(x) = p(y=1|x,θ)$ 

4. 为良性肿瘤的概率是: **$h_θ(x) = p(y=1|x,θ)$**，   
为恶性肿瘤的概率就可以直接得到: **p(y=0|x,θ) =1-p(y=1|x,θ)$**
因为 一个肿瘤不是良性就是恶性。


## 3. 决策边界(decision boundary)
### 1.什么是决策边界
<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200318-logistics-regression2.jpg" width="200"> 

假设 $ h_θ(x) =  g(θ^T x) = g(θ_0+θ_1x_1+θ_2x_2)$

$ -3 + x_1+x_2 =0$ 这条线，将样本分为两个区域，右上是y=1的样本，左下是y=0的样本。这条先叫做**决策边界**。   

当 $ -3 + x_1+x_2 =0$ 的时候$$ h_θ(θ^T x)$$ 等于0.5

### 2. 非线性决策边界
<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200318-logistics-regression3.jpg" width="300"> 

上面这个例子中 $-1 + x_1^2+x_2^2 =0$ (一个圆) 是决策边界。   
当$-1 + x_1^2+x_2^2 >0$(在圆外面) 是**y=1**的样本，当$-1 + x_1^2+x_2^2 <0 (在圆里面)$是**y=0**的样本


**逻辑回归从某种意义上来说 就是通过选择向量 $θ^T$ 来找到 决策边界**


## 4. 逻辑回归的代价函数(cost function)
### 1. 问题描述

<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200316-regression7.jpg" width="400"> 

问题是:**怎么 通过算法 从训练集找到合适的 $θ^T$?**
### 4. 代价函数
1. 线性回归(linear regression)代价函数
使用的是 **平方误差代价函数**   

$ J(θ) = \frac{1}{2n}\sum_{i=1}^n(h_θ(x^{(i)}) - y^{(i)} )^2  $



### 2. 为什么逻辑回归不能使用 线性回归 作为代价函数。   
因为 **sigmoid函数( $g(z) = \frac1 { 1+e^{-z}} $)** 是一个非线性的。 J(θ) 会是一个**非凸函数(non-convex function)**, 会有很多的局部最小值，不能保证收敛到全局最小值。
如下图所示：

<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200318-logistics-regression4.jpg" width="200"> 