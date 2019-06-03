---
layout: post
title:  mysql insert ignore 实现一个幂等操作
date:   2018-5-19 14:32:00
categories: 限流 高并发
---

###一、背景
"单词pk"有一个逻辑是，每一盘对战会消耗5个energy, ，在对战结束后会给赢家返还赢家5个energy。
对战结束后会到页面result页面，这个页面会进行返还energy。这里需要做一个"幂等"的处理，如果不做的话，用户多次刷新这个页面，就会多次返还energy。
这里使用了一个audit表，来做幂等。
```

CREATE TABLE `rank_audits` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `challenge_id` int(11) NOT NULL,
  `state` int(11) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `challenge_id` (`challenge_id`)
) ENGINE=InnoDB AUTO_INCREMENT=50 DEFAULT CHARSET=utf8mb4;

```
**challenge_id表示对战的id，唯一。**
每次访问result会尝试插入一条数据，如果成功则返还energy。

### 二、我以前的逻辑
 
 1. 错误的代码
 ```
    void audit(long challengeId, int activitiId, RankUserInfo userInfo, RankLevel level) {
        RankAudit ra = rankAuditService.getByChallengeId(challengeId);
        
        //如果有记录 表明已经返还了 energy，返回
        if (ra != null) {
            return;
        }
        
        ra = new RankAudit();
        ra.setChallengeId(challengeId);
        ra.setState(1);
        rankAuditService.insert(ra);

        //返还能量
        rankUserInfoService.addEnergy(activitiId, userInfo, level.getEnergyCost());
        //升级
        rankUserInfoService.levelUp(activitiId, userInfo, level.getMaxStar());
    }
```

这段代码是不优雅的。因为在并发操作下，有可能有两个请求同事执行下面这段代码，而且ra都为空

```
        RankAudit ra = rankAuditService.getByChallengeId(challengeId);
        //如果有记录 表明已经返还了 energy，返回
        if (ra != null) {
            return;
        }
```


当执行到下面的代码时候

```
 rankAuditService.insert(ra);
```
会因为challenge_id 是unique key而抛异常。



### 三、下面是改进后的代码
```
void audit(long challengeId, int activitiId, RankUserInfo userInfo, RankLevel level) {

        RankAudit ra = new RankAudit();
        ra.setChallengeId(challengeId);
        ra.setState(1);

        //插入成功表示可以返还能量
        if(rankAuditService.insertIgnore(ra) != 0){ 
            rankUserInfoService.addEnergy(activitiId, userInfo, level.getEnergyCost());
            rankUserInfoService.levelUp(activitiId, userInfo, level.getMaxStar());
        }
    }
```
关键在
rankAuditService.insertIgnore(ra) ，这个对应的sql是

```
 @Insert("insert ignore `rank_audits` (`id`,`challenge_id`,`state`,`created_at`) " +
            " values(NULL,#{ra.challengeId},#{ra.state},now());")
    @Options(useGeneratedKeys = true, keyProperty = "ra.id", keyColumn = "id")
    int insertIgnore(@Param("ra") RankAudit ra);
```

这段代码用了 insert ignore。如果插入成功返回1，执行  
```
rankUserInfoService.addEnergy(activitiId, userInfo, level.getEnergyCost());
rankUserInfoService.levelUp(activitiId, userInfo, level.getMaxStar());
```

返还energy和升级。如果插入返回 0表示已经返还过energy了，不做任何操作了。也不会抛异常。
这样会更优雅








 
    