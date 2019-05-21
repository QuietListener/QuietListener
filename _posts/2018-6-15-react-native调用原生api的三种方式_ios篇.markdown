---
layout: post
title:  react-native 调用原生api的三种方式(ios篇)
date:   2018-6-5 14:32:00
categories: react-native
---

# react-native 调用原生api的三种方式(ios篇)

做RN项目，常常需要访问一个原生系统的api又找不到相应的库，就需要自己写代码访问原生api。react-native提供了一个Bridge机制让javascript调用objective-c或者swift的原生方法。

我在做阅读项目过程中，第三方广告公司需要客户端提供用户的[IMSI码](https://www.baidu.com/link?url=rFAmKzcBRS3C6kBsD_XpYtYR0t1x3Snn63lG0clUYuYLK1LbtRe6jlrAtoqnNf8B_Zelkbb94L10frdlKWlrcq&wd=&eqid=b696b0d800004cce000000065b21f729) ,但是没有找到对应的rn库，就只能自己写一个库~。
# 一、Native端代码(objective-c）:
   在code项目中，新建一个文件教myNativeLib.m的文件，原生代码在这个文件实现。
   
   ![snapchat_xcode.png](https://upload-images.jianshu.io/upload_images/12693848-0f1c8bb0816a6cff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
    #import <Foundation/Foundation.h>
   #import "React/RCTLog.h"
   #import <React/RCTConvert.h>
   #import <React/RCTBridgeModule.h>
   
   //定义类
   @interface MyNativeLib : NSObject<RCTBridgeModule>
   @end
   
   //类实现
   @implementation MyNativeLib
   //在javascript端使用时候module名字叫MyNativeLibModule,如果参数为空，会默认使用类名
   RCT_EXPORT_MODULE(MyNativeLibModule)
   
   //相当于一个 js module export出去的方法 ,在js中使用RCTPromiseResolveBlock().then().catch()调用
   RCT_EXPORT_METHOD(getIMSIPromise: (RCTPromiseResolveBlock)resolve  rejecter:(RCTPromiseRejectBlock)reject    )
   {
   业务逻辑
   }
   }
   @end
```
## 下面分析一下这个类
#### 1.
   上面实现了一个类类名为MyNativeLib，这个类必须继承自 NSObject<RCTBridgeModule>，RCTBridgeModule这是RN框架提供的。
   
#### 2.
在JS端，RN会自动实例化类MyNativeLib，并且可以用下面的JS代码获取MyNativeLib的实例
```
 
import {NativeModules,NativeEventEmitter} from "react-native"
const MyNativeLibModule = NativeModules.MyNativeLibModule 
```
#### 3. 类名
下面这行代码决定了在js段使用native端实例的名字
```
RCT_EXPORT_MODULE(MyNativeLibModule)
```
 
在JS端使用"MyNativeLibModule"作为实例名
``` 
 
import {NativeModules,NativeEventEmitter} from "react-native"
const MyNativeLibModule = NativeModules.MyNativeLibModule 
```

如果不提供名字默认会用类名，例如RCT_EXPORT_MODULE()
在JS端就是用类名”MyNativeLib“获取实例 ：

```
import {NativeModules,NativeEventEmitter} from "react-native"
   const MyNativeLib = NativeModules.MyNativeLib 
```
   
#### 4.方法
   暴露给JS的方法需要使用一个宏 "RCT_EXPORT_METHOD" 包装起来。宏里面是一个普通Objective-c方法。方法的参数：resolve和reject后面会讲。
```
   RCT_EXPORT_METHOD(getIMSIPromise: (RCTPromiseResolveBlock)resolve  rejecter:(RCTPromiseRejectBlock)reject    )
   {
   业务逻辑
   }
```
   在js端我们就可以使用 下面代码 调用原生方法了。
   
```
   import {NativeModules,NativeEventEmitter} from "react-native"
   const MyNativeLibModule = NativeModules.MyNativeLibModule
   
   MyNativeLibModule.getIMSIPromise()
   .then(val=>{console.log(val)})
   .catch(e=>{console.error(e)})
```
   
## 二、3种调用原生api的方法
   下面假设我们有三个需求
   
* 1.获取app的版本
* 2.获取IMSI码
* 3.请求网络获取结果(假设很耗时需要3秒)
   
### 1 callback回调的方式获取结果
对于需求1,使用callback的方式实现。
>1.获取app的版本

##### 在js端，假设使用方法下面的代码获取结果。
```
   import {NativeModules,NativeEventEmitter} from "react-native"
   const MyNativeLibModule = NativeModules.MyNativeLibModule
   
   MyNativeLibModule.getAppVersionCallback((val,e)=>{
   if(e)
   console.error(e);
   else
   alert(`成功获取AppVersion:${val}`) 
   };
   }) ;
   
```
   
Js端使用一个匿名回调函数来获取结果
```
   (val,e)=> {
   if(e)
   console.error(e);
   else
   alert(`成功获取AppVersion:${val}`) 
   };
```
   
   
##### 下面我们看看原生的Objective-c代码怎么写：
```
   RCT_EXPORT_METHOD(getAppVersionCallback:(RCTResponseSenderBlock)callback)
   {
   NSString * version = nil;
   @try {
   //假设version是1.0.5
   version = @"1.0.5";
   
   //callback只接受一个数组做参数
   callback(@[version,[NSNull null]]);
   } @catch (NSException *exception) {
   callback(@[version,exception]);
   }
   }
```
分析这个函数
* 1.RCT_EXPORT_METHOD宏表示是要暴露给JS的方法。
* 2.getAppVersionCallback 中，有一个类型为RCTResponseSenderBlock的参数callback ，用于回调。它方法只接受一个数组作为参数。
   
   
### 2 promise的方式获取结果

使用callback是可行的，对于ES6 有了 promise，ES7有了async/await，我们也可以使用的。对于需求2使用promise来实现
>2.获取IMSI码

##### JS代码
```
   //使用promise
   MyNativeLibModule.getIMSIPromise().then(val => {
   alert(`成功获取IMSI:${val}`)
   }).catch(e => {
   console.error(e);
   })
   ```
   ##### Objective-c代码
   ```RCTPromiseResolveBlock().then().catch()调用
   RCT_EXPORT_METHOD(getIMSIPromise: (RCTPromiseResolveBlock)resolve  rejecter:(RCTPromiseRejectBlock)reject    )
   {
   NSString * imsi = nil;
   @try {
   imsi = @"46000";
   resolve(imsi);
   
   } @catch (NSException *exception) {
   reject(@"", @"失败", nil);
   }
   }
```
getIMSIPromise中提供的2个参数  rejecter，reject与JS仲的resolve和reject一一对应。
   
   
### 3 耗时的方法，放入queue仲去执行 
   上面两个例子耗时都很少，瞬间返回。有的方法很耗时，比如需求 

>3.请求网络获取结果(假设很耗时需要3秒)。

这时候我们需要执行一个异步逻辑。IOS中要执行的逻辑可以放到一个队列中去执行~
   
#### JS端方法(同样使用promise的方式)
```
   MyNativeLibModule.getResponsePromise().then(val => {
   alert(`成功获取response(非常耗时):${val}`)
   }).catch(e => {
   console.error(e);
   })
   
```
   
#### native 端代码
```
   //很耗时的方法,使用异步 ,在js中使用RCTPromiseResolveBlock().then().catch()调用
   RCT_EXPORT_METHOD(getResponsePromise: (RCTPromiseResolveBlock)resolve  rejecter:(RCTPromiseRejectBlock)reject    )
   {
   //获取ios的global_queue
   dispatch_queue_t myQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
   
   //业务逻辑放入queue仲执行
   dispatch_async(myQueue, ^{
   @try {
   NSString * response = nil;
   /*假装获取网络资源需要3秒钟*/
   sleep(3);
   
   response = @"response is {data:123}";
   resolve(response);
   
   } @catch (NSException *exception) {
   reject(@"", @"注册失败", nil);
   }
   });
```
   
#### 下面分析一下上面的代码
##### 1.获取IOS中默认的globalQueue，用于执行异步逻辑
```
   dispatch_queue_t myQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```
   
   
##### 2. dispatch_asyn将业务逻辑放到myQueue中，并且异步执行，业务逻辑是一个block，如下所示
```
   ^{
   @try {
   NSString * response = nil;
   /*假装获取网络资源需要3秒钟*/
   sleep(3);
   
   response = @"response is {data:123}";
   resolve(response);
   
   } @catch (NSException *exception) {
   reject(@"", @"注册失败", nil);
   }
   }
```
这样会在调用，后立即返回，不会阻塞UI

```
   //调用后立即返回，不会阻塞
   MyNativeLibModule.getResponsePromise().then().catch()
```
   
### Native端的源代码在：
#### https://github.com/QuietListener/rn_demo_1/blob/master/ios/rn_demo_1/myNativeLib.m
   
### JS端的源代码在： 
#### https://github.com/QuietListener/rn_demo_1/blob/master/App.js
