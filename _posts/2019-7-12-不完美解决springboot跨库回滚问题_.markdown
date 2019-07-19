---
layout: post
title:  不完美解决springboot跨库回滚问题
date:   2019-7-19 14:32:00
categories: springboot 分布式事务
---

### 1. 背景
业务上有一个补打卡功能，流程是
1. 先检查是否补打卡
2. 扣1500金币(数据库1)
3. 打卡操作(数据库2)

由于这个功能与**金币**挂钩，金币需要购买，也就与钱挂钩了，所有对一致性有要求。
第2步金币和第3步打卡数据分别在不同的库中，所以这里存在一个跨库事务的问题。
例如扣金币成功，打卡失败，需要两个跨库事务回滚。springboot的@Transactional注解功能，默认只能回滚一个库。

### 2. 解决办法
我们只能手动指定TransactionManager，并手动commit(或者rollback)所有的事务了。我封装了一个方法**manualTransaction**来实现这个功能
```
package xxx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.function.Supplier;

@Service
public class CommonService {
    @Autowired
    private ApplicationContext applicationContext;

    /**
     *  txManagerNames: TransactionManager的名字
     *  supplier : 业务逻辑
     */
    public <T> T manualTransaction(List<String> txManagerNames,Supplier<T> supplier) {
        List<TransactionStatus> statues = new ArrayList<>();
        List<PlatformTransactionManager> managers = new ArrayList<>();
        for (String name : txManagerNames) {
            //获取TransactionManager
            DataSourceTransactionManager manager = (DataSourceTransactionManager) applicationContext.getBean(name);
            DefaultTransactionDefinition def = new DefaultTransactionDefinition();
            //事务状态
            TransactionStatus status = manager.getTransaction(def); 
            managers.add(manager);
            statues.add(status);
        }

        boolean flag = false;
        try {
            //执行业务逻辑
            T t= supplier.get();
            flag = true;
            return t;
        } finally {
            //要么全部回滚，要么全部提交
            //注意mannager的顺序要倒过来
            for (int i = managers.size() - 1; i >= 0; i--) {
                PlatformTransactionManager manager = managers.get(i);
                TransactionStatus status = statues.get(i);
                if (flag == true) {
                    manager.commit(status);
                } else {
                    manager.rollback(status);
                }
            }
        }


    }
}

```
**下面是使用业务中所使用**  

**“transactionManager1” 和 “transactionManager2”** 分别是数据库1和数据库2的TransactionManager对应的Bean的名字    
**() -> dakaCredit_(userId, timeStampS, clientTimeS, timezone))** 是业务逻辑(Supplier)
```
/**
     *
     * @param userId
     * @param timeStampS
     * @param clientTimeS
     * @param timezone
     * @return # flag: 成功(0), 补失败(1), 扣铜板失败(2)
     */
    public int dakaCredit(long userId, long timeStampS, int clientTimeS,int timezone) {
        List<String> txManagerNames = Arrays.asList(“transactionManager1”,"“transactionManager2”");
       
        int ret = commonService.manualTransaction(txManagerNames,
                () -> dakaCredit_(userId, timeStampS, clientTimeS, timezone)); //需要同时成功，同时失败的业务逻辑
        return ret;
    }

     /**
       *补打卡功能
       */
     private int dakaCredit_(long userId, long timeStampS, int clientTimeS,int timezone) {

        Map<String,Long> extra = new HashMap<>();
        extra.put("timestamp",timeStampS);
        extra.put("timezone",(long)clientTimeS);

        Date dakaDate = new Date(timeStampS*1000);
        UserCreditDakaDates ucd = new UserCreditDakaDates();
        ucd.setUserId(userId);
        ucd.setDakaDate(dakaDate);
        ucd.setDakaTimestamp(timeStampS);
        ucd.setDakaTimezone(timezone);

        //业务逻辑
        userCreditDakaDatesMapper.insertOrUpdate(ucd);
        String orderId = generateOrderId(userId);
        
        //扣金币，数据在库1中
        int ret1 = userCreditService.updateUserCredit(userId,-1500,orderId,3,gson.toJson(extra));
        if(ret1 == 0){
            return 2;
        }
        //补打卡  数据在库2中
        daka(userId, dakaDate, clientTimeS);
        return 0;
    }
```

**注意事项:** 
需要回滚的业务逻辑的Propagation必须为Propagation.REQUIRED，业务逻辑能复用manualTransaction生成的事务，如果是Propagation.REQUIRES_NEW的话就不行，因为每次都重新生成一个新事务，这个新事物就不在manualTransaction的管理范围了。


### 3.缺点
  很明显这个方案是有很大的缺点的。看下面的代码片段
  ```
     try {
            //执行业务逻辑
            T t= supplier.get();

            flag = true;
            return t;
        } finally {
            //要么全部回滚，要么全部提交
            for (int i = managers.size() - 1; i >= 0; i--) {
                PlatformTransactionManager manager = managers.get(i);
                TransactionStatus status = statues.get(i);
                if (flag == true) {
                    manager.commit(status);
                } else {
                    try{
                        manager.rollback(status);
                    }catch(Exception e){
                        logger.error("rollback error",e);
                    }
                }
            }
        }
  ```
  在finally中 可有可能发生下面的情况.
  在flag=true的条件下所有事务都应该commit，但是有些commit失败了，有些成功了，会导致数据不一致。
  但是考虑到业务场景是一个比较低频的操作，而且commit失败的概率也是非常的小，还有金币不是真正的钱，对用户不敏感业务条件下。我们可以记录详细的log，如果出问题也可以从log中来恢复。这也算是一个不完美的解决方案了。
     
### 4.其他方案1 XA分布式事务
   可以使用 XA协议，实现跨库事务。 atomikos这个库实现了jta的功能。但是使用atomikos需要将DataSource替换为AtomikosDataSourceBean(原来的DataSource是com.zaxxer.hikari.HikariDataSource)，对项目影响比较大，也没有充分的时间进行测试。而且XA的效率是单机事务的1/10,应对突发流量方面会有比较大的瓶颈。所以不采用这种方案。
### 5.其他方案2 最终一致
  也可以使用最终一致的方法，扣款成功，但是打卡失败，将信息写一个表,然后定期重试(补偿的方案)。但是如果重试一定次数失败，还得做一些回滚操作。比如归还给用户的金币。这大大增加了业务逻辑的复杂度，而且这是一个低频接口，没有必要。

### 参考资料:
   1. [使用JTA处理分布式事务](https://www.hifreud.com/2017/07/12/spring-boot-23-jta-handle-distribute-transaction)
   
   2. [详解Mysql分布式事务XA](https://blog.csdn.net/soonfly/article/details/70677138 )
