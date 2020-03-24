---
layout: post
title: 吴恩达机器学习记录6 逻辑回归实战(octave)
date: 2020-3-23 14:32:00
categories:  机器学习
---
## 1. 数据

数据为两列，第一列为肿瘤大小，第二列为肿瘤类型：1表示是恶性肿瘤，0表示良性肿瘤

```java

data = load("lr_data_cancer.txt");
plot(data(:,1),data(:,2),"rx",'MarkerSize', 10);
```

<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200323-lr-octave5.jpg" width="500"> 


### 2. logistic regression 进行训练。
#### 1. 梯度下降函数
```java

function [theta,history_theta] = gradientDescentLr(X, y, theta, alpha, iterations)  
%X为训练数据的输入值，y为标记的值，alpha为梯度下降的速率，  iterations为迭代次数
m = length(y); %训练数据个数
history_theta = zeros(m,2); %记录theta的变化
for i = 1:iterations %更新theta
  h = (1./(1+exp(-1*X * theta)) .- y);
  theta(1) = theta(1) - alpha / m * sum(h);       
  theta(2) = theta(2) - alpha / m * sum(h.* X(:,2));
  %disp(sprintf('i= %d (%0.2f,%0.2f)',i,theta(1),theta(2)))

  %记录当前theta 
  history_theta(i,1) = theta(1);
  history_theta(i,2) = theta(2);
end
```

**执行代码**
```java
>> y = data(:,2);
>> m = length(y);
>> X = [ones(m,1),data(:,1)];
>>
>> alpha = 0.01;
>> iterations = 3000;
>> theta = [0;0];
>>
>> [theta,history_theta] = gradientDescentLr(X, y, theta, alpha, iterations);
>> theta;
>> x = linspace(0,8,100);
>> y = 1./(1+exp(-1*(theta(1) + theta(2)*x )));
>> hold on;
>> plot(x,y,"-");
>> theta;
>> theta
theta =

  -2.5002
   1.2758

>>
```

<img src="https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/20200323-lr-octave6.jpg" width="500"> 
