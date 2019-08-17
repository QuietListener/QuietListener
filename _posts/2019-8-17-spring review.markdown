---
layout: post
title:  spring review
date:   2019-8-17 14:32:00
categories: java web servlet
---
# 1.写在前面
各种review，今天轮到是spring。

# 2.why spring 
## 1. Pojo非常轻量级，非侵入
 有些框架，需要你继承他的类，侵入式的，spring不会强迫实现Spring规范的类

## 2.依赖注入(DI)和面向接口，实现IOC(控制反转)，最终实现松耦合
### 1.传统做法(高耦合)
传统的做法是每个对象自己管理与自己协作的对象，比如下面，这样 耦合性就非常高。而且测试也会不方便。
```java
  public class A {
      private B b = null;
      public A(){
          //A与B耦合，A依赖B。
          //测试的时候，也无法替换A为MockA
          this.a = new A(); 
      }

      public void func(){
          this.a.func()
      }
  }
```
### 3. Spring作为第三方，使用DI(Dependency Inject)来装配对象. 
这样A不用关系B是如何初始化的，由第三方(spring)调用setB来组装(wire)A
原来使用this.b= new B(),是B是受A控制的，但是现在使用DI,b是由第三方注入的，不受A控制了，这就是IOC(控制反转)
```java
public class A {
      private B b = null;

      public A(){
      }

      public void func(){
          this.a.func()
      }
      public void setB(B b){
          this.b = b;
      }

      public B getB(){
        return this.b;
      }
  }
```

### 4. AOP（Aspect Oriental Programming）面向切面编程
#### 1.如果不使用AOP，很多耦合很紧
下面 日志功能,鉴权功能和业务逻辑耦合在一起
```java
public class StuffController extend HttpServlet{

    //日志功能和鉴权功能与 业务耦合在一起，
    private Logger logger; //日志
    private Auth auth; //鉴权

    public void doGet(HttpServletRequest req,HttpServletResponse res){

        //日志功能和鉴权功能与 业务耦合在一起，
        logger.info(new Date().toString() + req.getURI());
        if(!auth.authenticate(req)){
            logger.error(new Date()+"error"); 
            return ;
        } 
        //下面是业务逻辑
    }
    ....
}

```
#### 2.使用AOP让业务只关心自己的核心功能。一些通用的功能可以在切面中实现

下面是使用AOP技术改造后的代码。
1. 定义了一个AuthFlag注解。
2. 定义了一个AuthAop,@Around("@annotation(AuthFlag)")表示添加了@AuthFlag的方法将会使用到 方法advice的功能。
3. StuffController只需要在doGet方法上添加@AuthFlag就拥有了鉴权的功能。
这样 Auth和业务逻辑耦合就非常的小，而且减少了很多重复代码。


```java
//定义一个鉴权的注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthFlag {
}


//鉴权切面
@Aspect
@Component
public class AuthAop {

    private Auth auth = new Auth();
   
    @Around("@annotation(AuthFlag)")
    public Object advice(ProceedingJoinPoint joinPoint) {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Object[] params = joinPoint.getArgs();
      
        HttpServletRequest req = (HttpServletRequest)params[0];
        //在切面中实现鉴权
        if(!auth.authenticate(req)){
            return ;
        } 

        joinPoint.proceed();
    }
}

public class StuffController extend HttpServlet{

    //只需要添加一个注解 @AuthFlag，就可以实现鉴权功能。
    @AuthFlag
    public void doGet(HttpServletRequest req,HttpServletResponse res){
        //下面是业务逻辑
    }
   
}

```


# 3. 用Spring来装配对象

## 1.BeanFactory和ApplicationContext
   BeanFactory是一个接口，顾名思义，就是对象工厂，Spring中的Bean就是由它来装配的。这是Ta比较低级，ApplicationContext继承了BeanFactory，提供了很多额外的功能。一般使用ApplicationContext

## 2. ApplicationContext的几个常用实现
    1. AnnotationConfigApplicationContext 从一个或多个基于java的配置类中加载Spring上下文。
    2. AnnotationConfigWebApplicationContext从一个或多个基于java的配置类中加载Spring Web应用的上下文。
    3. ClassPathXmlApplicationContext从类路径下的一个或者多个xml配置中加载上下文
    4. FileSystemXmlApplicationContext从文件系统下的一个或者多个xml配置中加载上下文
    5. XmlWebApplicationContext从web应用下的一个或者多个xml配置中加载上下文

