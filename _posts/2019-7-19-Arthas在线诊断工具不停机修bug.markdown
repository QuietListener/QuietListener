---
layout: post
title:  Arthas在线诊断工具不停机修bug
date:   2019-7-12 14:32:00
categories: jvm
---
Arthas是阿里巴巴开源的诊断工具


### 1.安装与使用见
[https://alibaba.github.io/arthas/watch.html](https://alibaba.github.io/arthas/watch.html)

### 2. watch命令 解决一个线上问题
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



### 3. trace命令找到性能瓶颈
测试报了一个问题，上线看是rpc服务调用超时了，应该rpc有性能瓶颈。
最后定位到是  StudyRecordService queryByUserIdAndTopicIds 这个方法rpc调用。
使用 arthas在测试服务器测试看看瓶颈在哪里?
```java
[arthas@28672]$ trace -n 1  bcz.app.server.study.record.service.StudyRecordService queryByUserIdAndTopicIds  
Press Q or Ctrl+C to abort.
Affect(class-cnt:2 , method-cnt:2) cost in 81 ms.
`---ts=2020-03-10 11:02:36;thread_name=http-nio-7002-exec-2;id=2e;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@84a7c30
    `---[4595.080606ms] bcz.app.server.study.record.service.StudyRecordService$$EnhancerBySpringCGLIB$$8d9f16f8:queryByUserIdAndTopicIds()
        `---[4594.890413ms] org.springframework.cglib.proxy.MethodInterceptor:intercept() #0
            `---[4593.941665ms] bcz.app.server.study.record.service.StudyRecordService:queryByUserIdAndTopicIds()
                +---[0.017715ms] org.springframework.util.CollectionUtils:isEmpty() #64
                +---[1776.405382ms] bcz.app.server.study.record.service.UserDoneWordService:queryByUserIdAndTopicIds() #65
                +---[1685.499361ms] bcz.app.server.study.record.service.UwExtraService:queryByUwIds() #66
                +---[min=1.22E-4ms,max=0.033878ms,total=53.844545ms,count=213354] bcz.app.server.study.record.service.bo.StudyRecord:<init>() #72
                +---[min=1.25E-4ms,max=0.498781ms,total=57.830216ms,count=213354] bcz.app.server.study.record.service.bo.StudyRecord:setUserDoneWord() #73
                +---[min=1.23E-4ms,max=0.015333ms,total=54.945995ms,count=213354] bcz.app.server.study.record.dao.domain.UserDoneWord:getId() #74
                +---[min=1.23E-4ms,max=0.015893ms,total=5.346287ms,count=18226] bcz.app.server.study.record.dao.domain.UwExtra:getExtra() #75
                +---[min=1.26E-4ms,max=0.022879ms,total=5.408262ms,count=18226] org.springframework.util.StringUtils:isEmpty() #75
                +---[min=1.26E-4ms,max=0.01338ms,total=5.151019ms,count=18226] bcz.app.server.study.record.dao.domain.UwExtra:getExtra() #76
                +---[min=8.2E-4ms,max=33.627127ms,total=62.030956ms,count=18226] com.google.gson.Gson:fromJson() #76
                `---[min=1.3E-4ms,max=0.014048ms,total=5.488352ms,count=18226] bcz.app.server.study.record.service.bo.StudyRecord:setExtraScore() #77

```

最后发现是这两个方法需要优化，每个都耗时操作1500ms。
```
  +---[1776.405382ms] bcz.app.server.study.record.service.UserDoneWordService:queryByUserIdAndTopicIds() #65
  +---[1685.499361ms] bcz.app.server.study.record.service.UwExtraService:queryByUwIds() #66

```