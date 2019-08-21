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

**这样 Auth和业务逻辑耦合就非常的小，而且减少了很多重复代码**


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



# 4. 装配Bean
## 1. spring有三种装配方式
     1. 自动装配
     2. JavaConfig装配
     3. xml装配
使用得比较多的是自动装配

##  2. 自动装配  
   自动装配使用注解(Annotation)进行装配，主要涉及到的注解有

```java

@Configuration
@Component
@ComponentScan
@AutoWire
@Primary
@Qualifier
@Scope
@LookUp
@Import
```

### 1. @Component
@Component表明该类会作为组件，Spring会为这个类创建bean ，还可以使用value为主键指定一个名字

```java
@Component(value = "BDisk")
public class TypeBDisk implements Disk {
    private int rand = 0;
    public TypeBDisk(){
        this.rand = new Random().nextInt(100000);
    }

    @Override
    public void play() {
        System.out.println(this.getClass().getName()+this.rand);
    }
}

```


### 2. @Configuration
Configuration 表明这个类是一个配置类，该类应该包含在Spring上下文中如何创建Bean的细节

### 3. @ComponentScan
自动扫描并将带有@Component注释的类创建为对应的Bean   
可以通过basePackages指定要扫描的包名
```java
@ComponentScan(basePackages = { "andy.com.springFramework.core.basic.wire.autowire.pb"})
public class AutoConfig1 {
}
```
### 4. @AutoWire @Qualifier
@AutoWire 表示自动注入，可以使用在属性上，required=false表示spring上下文中没有需要的实例，不报错。
@Qualifier("BDisk")指定名字为"BDisk"的实例

```java
@Component
public class Player {

    @Autowired(required = true)
    @Qualifier("BDisk")
    private Disk disk;

    @Autowired
    private TypeADisk adisk;

}
```
### 5.@Primary
当一个接口有多个实现的时候，使用@Primary的会被优先选择
例如 disk有多个实现，会选择，在Player中会选择有@Primary注解的实例(TypeADisk)。
```java

@Component
public class Player {


    @Autowired(required = true)
    private Disk disk;
}

@Component
@Primary //当有多个disk实例时候优选选择
public class TypeADisk implements Disk {
    @Override
    public void play() {
        System.out.println(this.getClass().getName());
    }
}

@Component
public class TypeBDisk implements Disk {
    @Override
    public void play() {
        System.out.println(this.getClass().getName());
    }
}


```


### 6. @Scope和@LookUp
#### 1.Scope
spring默认Bean的scope有singleton和prototype，默认是singleton(单例)的。
singleton每次都使用同一个实例，prototype表示每次都新初始化一个新实例。

```java

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //每次都创建一个新的
public class TypeADisk implements Disk {
    @Override
    public void play() {
        System.out.println(this.getClass().getName()+rand);
    }
}

```

#### 2. 一个的Singleton实例引用一个Prototype的实例引发的问题
当一个Singleton的实例引用一个Prototype的时候，会发现每次还是使用的同一个Prototype实例，因为Singleton的类只会实例化一次。
例如下面的代码 虽然TypeADisk是Prototype的但是，Player是单例，每次都使用的同一个TypeADisk实例。

```java

@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //每次都创建一个新的
public class TypeADisk implements Disk {
    @Override
    public void play() {
        System.out.println(this.getClass().getName()+rand);
    }
}


@Component
public class Player {

    //TypeADisk是Prototype的但是每次都使用同一个TypeADisk实例
    @Autowired
    private TypeADisk adisk;

    public void playa(){
        adisk.play();
    }
}
```

#### 3. 使用@Lookup解决上面的问题
@Lookup只能使用在方法上，每次调用@Lookup修饰的方法，spring都会返回一个新的实例。

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) //每次都创建一个新的
public class TypeADisk implements Disk {
    @Override
    public void play() {
        System.out.println(this.getClass().getName());
    }
}

@Component
public class Player {

    public void playa(){
        getAdisk().play();
    }

    @Lookup
    public TypeADisk getAdisk(){
        return null;
    }
}
```

### 7. @Import
@Import用来导入其他的@Configuration配置过的bean
例如
```java
@Configuration
//扫描带有@Component的类，并将其创建一个Bean
@ComponentScan(basePackages = { "andy.com.springFramework.core.basic.wire.autowire.pa"})
public class AutoConfig {
}

@Configuration
@Import({AutoConfig.class})
public class AutoConfigAll {
}
```
### 8. 使用AnnotationConfigApplicationContext来开始Spring应用程序

```java
 public void tets(){
        ApplicationContext context = new AnnotationConfigApplicationContext(AutoConfigAll.class);

        TypeADisk aDisk = context.getBean(TypeADisk.class);
        TypeBDisk bDisk = context.getBean(TypeBDisk.class);
 }
```


##  2. 使用Java代码装配

### 1. 使用java代码装配的场景
有的时候我们没有源代码，不能使用 @Component和@AutoWire 来自动装配，比如第三方库，这时候就需要使用java代码来装配。或者使用java代码装配更方便的地方

### 2.在方法上使用@Bean来装配

#### 1.代码
```java
/**
 * 播放机
 */
public class CdPlayer {
    private Disc disc;

    public Disc getDisk() {
        return disc;
    }

    public void setDisk(Disc disc) {
        this.disc = disc;
    }
}

/**
 * CD
 */
public class Disc {

    private String name;
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}



/**
 * 装配类
 */
@Configuration
public class JavaConfig {

    @Bean(name="disc")
    public Disc getDisc(){
        return new Disc();
    }

    @Bean(name="player1")
    public CdPlayer newCdPlayer(){
        CdPlayer player = new CdPlayer();
        //当调用getDisc时候，spring会拦截返回同一个Disc对象
        player.setDisk(getDisc());
        return player;
    }

    /**
     * 结合@AutoWire在参数上来装配
     * @param disc
     * @return
     */
    @Bean(name="player2")
    public CdPlayer new1CdPlayer(@Autowired Disc disc){
        CdPlayer player = new CdPlayer();
        player.setDisk(disc);
        return player;
    }
}


```
#### 2.测试

```java
    @Test
    public void tets(){
       
        ApplicationContext context = new AnnotationConfigApplicationContext(JavaConfig.class);
        Disc d1 = context.getBean(Disc.class);
        Disc d2 = context.getBean(Disc.class);

        //使用同一个Disc对象
        assert d1 == d2;

        CdPlayer player1 = (CdPlayer)context.getBean("player1");
        CdPlayer player2 = (CdPlayer)context.getBean("player2");

        //使用同一个Disc对象
        assert player1.getDisk() == player2.getDisk();
        assert d1 == player1.getDisk();
    }
```    


## 3.使用XML来配置，(xml用得很少了)。


## 4. 混合配置
### 1. JavaConfig配置导入xml配置
common.xml 是一个spring的xml配置文件。
```java

@Configuration
@Import({AnotherConfig.class})
@ImportResource("classpath:common.xml") //common.xml在项目的classpath中
public class JavaConfig {

    @Bean(name="disc")
    public Disc getDisc(){
        return new Disc();
    }

    @Bean(name="player1")
    public CdPlayer newCdPlayer(){
        CdPlayer player = new CdPlayer();
        //当调用getDisc时候，spring会拦截返回同一个Disc对象
        player.setDisk(getDisc());
        return player;
    }

    /**
     * 结合@AutoWire在参数上来装配
     * @param disc
     * @return
     */
    @Bean(name="player2")
    public CdPlayer new1CdPlayer(@Autowired Disc disc){
        CdPlayer player = new CdPlayer();
        player.setDisk(disc);
        return player;
    }
}


```
### 2. xml中导入JavaConfig配置和其他xml配置
common.xml 是一个spring的xml配置文件。  
andy.com.springFramework.core.basic.wire.javaConfig.JavaConfig是一个有@Configuration的配置类

```xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="
         http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">


    <!--导入JavaConfig中的配置-->
    <bean class="andy.com.springFramework.core.basic.wire.javaConfig.JavaConfig"></bean>

    <!--导入其他xml配置-->
    <import resource="classpath:common.xml"></import>

</beans>

```



### 3. @Profile切换环境
#### 1. why @Profile
在SpringContext初始化的时候@Profile可以决定组装哪些类。  
使用 -Dspring.profiles.active=xxx 参数来指定 要组装的bean。  
例如 -Dspring.profiles.active=prod，只会组装@Proifle("prod")修饰的方法(或者类)，不会实例化@Proifle("dev")等其他修饰的方法(或者类)    
##### 好处
**不用重新打包就能实现切换环境。**能保证测试的包和发布的包是同一个包，减少重新打包带来的风险。比如测试测好了，重新打包上线，新包可能会有一些偶然的变化，导致
测试好的代码也有上线失败的风险。

#### 1.代码与测试
```java

public class ProjectConf {
    private String name;

    public ProjectConf(String name){
        this.name = name;
    }

    public String getName() {
        return name;
    }
}


@Configuration
public class JavaConfig {

    @Bean
    @Profile("dev")
    public ProjectConf devConfig() {
        return new ProjectConf("dev");
    }

    @Bean
    @Profile("prod")
    public ProjectConf prodConfig() {
        return new ProjectConf("prod");
    }
}

public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(JavaConfig.class);
        ProjectConf projectConf = context.getBean(ProjectConf.class);

        /**
         *当使用 java -Dspring.profiles.active=prod  andy.com.springFramework.core.basic.wire.advance.JavaConfig --classpath=....测试时候输出*prod*
         */
        System.out.println(projectConf.getName());
    }
}

```

当使用 java -Dspring.profiles.active=prod  andy.com.springFramework.core.basic.wire.advance.JavaConfig --classpath=.... 测试时候输出**prod**


**也可以用在ServletContext中配置。**
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                     http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
        version="3.0"
        metadata-complete="true">

   <servlet>
       <servlet-name>ListenerServlet</servlet-name>
       <servlet-class>andy.com.demo.listener.ListenerServlet</servlet-class>
       <load-on-startup>20</load-on-startup>
   </servlet>

   <servlet-mapping>
       <servlet-name>ListenerServlet</servlet-name>
       <url-pattern>/listenerServlet</url-pattern>
   </servlet-mapping>

   <!-- 初始化ServletContext时候会用到 添加 spring.profiles.active-->
   <context-param>
       <param-name>spring.profiles.active</param-name>
       <param-value>prod</param-value>
   </context-param>

</web-app>

```






