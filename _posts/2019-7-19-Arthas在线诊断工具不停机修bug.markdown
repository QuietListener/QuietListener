---
layout: post
title:  Arthas在线诊断工具不停机修bug
date:   2019-7-12 14:32:00
categories: jvm
---
Arthas是阿里巴巴开源的诊断工具


### 1.安装与使用见
[https://alibaba.github.io/arthas/watch.html](https://alibaba.github.io/arthas/watch.html)

### 2. 解决一个线上问题
#### 1.问题
客户报了一个 拉取学习记录出错的问题，用户id为25957451,书id为299,看log发现是gson的错误.
```
com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was NUMBER at line 1 column 14 path $
```
原来是又学习记录有一个字段extra 存放的是json格式的数据，在获取的时候需要使用gson.from(extra,ExtraInfo.class)。猜想应该是extra有问题。

#### 2.疑惑
然后在数据库中找用户id为25957451,书id为299的学习记录信息。但是很不幸数据库记录为空，不会执行gson.from 啊。这让我百思不得其解了。
#### 3.使用Arthas去线上看(watch)一下
调用学习记录的方法是 bcz.app.server.study.record.service.UserDoneWordService.queryByUserIdAndBookId

去线上看一下到底是什么情况。
```
$ watch bcz.app.server.study.record.service.UserDoneWordService  queryByUserIdAndBookId  "{params,returnObj}" "params[0]==25957451"  -x 2
Press Q or Ctrl+C to abort.
Affect(class-cnt:2 , method-cnt:2) cost in 122 ms.
ts=2019-07-23 15:29:01; [cost=1.776722ms] result=@ArrayList[
    @Object[][
        @Long[25957451],
        @Integer[299],
    ],
    @ArrayList[isEmpty=true;size=0],
]
ts=2019-07-23 15:29:01; [cost=2.956089ms] result=@ArrayList[
    @Object[][
        @Long[25957451],
        @Integer[299],
    ],
    @ArrayList[isEmpty=true;size=0],
]
ts=2019-07-23 15:29:01; [cost=1.017795ms] result=@ArrayList[
    @Object[][
        @Long[25957451],
        @Integer[0],
    ],
    @ArrayList[
        @UserDoneWord[UserDoneWord{id=13108202, userId=25957451, bookId=0, wordTopicId=45, firstAt=774, score=1451710260, wrongTimes=6, doneTimes=0, totalUsedTime=6, createdAt=1970-01-01 08:30:38.0, updatedAt=2016-01-02 12:51:00.0}],
    ],
]
```

"params[0]==25957451" 表示只关心user_id为25957451的调用。
在结果中
```
 @Object[][
        @Long[25957451],
        @Integer[299],
    ],
 ```
表示用户id为25957451，书id为299这是正常的。返回为空也是正常的。

#### 4.找到问题所在
但是看下面的数据说明还使用参数，用户id为25957451，书id为0，来调用函数。返回结果有一个UserDoneWord。说明在获取书id为299的学习记录的同时还会去获取书id为0的记录。

```
  @Object[][
        @Long[25957451],
        @Integer[0],
    ],
    @ArrayList[
        @UserDoneWord[UserDoneWord{id=13108202, userId=25957451, bookId=0, wordTopicId=45, firstAt=774, score=1451710260, wrongTimes=6, doneTimes=0, totalUsedTime=6, createdAt=1970-01-01 08:30:38.0, updatedAt=2016-01-02 12:51:00.0}],
    ],
    
```

然后在认真读取源代码代码，发现为了兼容老用户(历史原因)，在获取某本书的学习记录的时候，还会去获取id为0的书的记录。这下找到了原因。也找到了有问题的学习记录。


