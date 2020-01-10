---
layout: post
title: spring-cloud spring-boot
date: 2019-12-24 14:32:00
categories:  java javascript
---
spring-boot也用过这么久了，总结一下~
# 1. 一个趋势
以前java Web服务很多都是将程序代码打包为jar，然后将部署到web容器，tomcat ，jetty等。现在spring-boot刚好反过来，将web容器(tomcat,jetty等) 打包进jar。 这样可以像执行一个普通jar包一样来发布web应用，也省去了以前那种每台机器都去配置一个tomcat的麻烦。

# 1. spring-boot好在哪里?
主要有三个方面
## 1. 自动配置
   常见的应用功能，spring自动提供相关配置
2. 起步依赖 starter
   告诉spring-boot需要什么功能，它就引入需要的库，并且这些库是经过spring-boot 开发团队测试过的，不会有问题。不用像以前一样配一个spring-mvc 满世界找依赖，找到了以后还可能有版本不兼容问题。
3. acuator
   了解spring运行的情况。



# 2. SpringBoot 的启动
## 1. 基本结构
SpringBoot启动只需要加上一个@SpringBootApplication注解即可
```java
@SpringBootApplication
public class SpringbootServiceSubmitApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringbootServiceSubmitApplication.class, args); //负责启动引导应用程序
    }
}
```
这样会编译后会生成一个jar包，直接在命令行运行。

## 2. @SpringBootApplication 的魔法

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

... 
}
```

**@SpringBootApplication = @ComponentScan + @EnableAutoConfiguration + @SpringBootConfiguration(就是@Configuration)**
1.  @ComponentScan 会扫描 SpringbootServiceSubmitApplication所在包的Spring配置。
2. @SpringBootConfiguration(@Configuration) 表明这是一个基于java配置文件
3. **@EnableAutoConfiguration**  这开启了springboot自动配置的魔法，少写几百行的配置。

# 2.依赖 
  排除一些不想有的依赖
  ```xml
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <!--不用tomcat的数据库连接池-->
            <exclusions>
                <exclusion>
                    <groupId>org.apache.tomcat</groupId>
                    <artifactId>tomcat-jdbc</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

# 2. 自动配置
  springboot自动配置在启动时候,才决定使用哪些配置，不是用哪些配置。
  比如: 
  1. JdbcTemplate是不是在ClassPath里？ 如果是，并且有DataSource的Bean，则自动配置一个JdbcTemplate   
  2. Spring Security是不是在Classpath里?如果是，则进行一个非常基本的Web安全设置

  >每当应用程序启动的时候，Spring Boot的自动配置都要做将近200个这样的决定，涵盖安全、 
  >集成、持久化、Web开发等诸多方面。所有这些自动配置就是为了尽量不让你自己写配置。


## 1. 原理 Condition接口+Conditional注解
Spring 4.0引入了  Condition接口和Conditional注解  可以实现根据某些条件来自实例化Bean
