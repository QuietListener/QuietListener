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
spring4.0 提供了**Condition接口**和 **@Conditional注解**
@Conditional需要一个实现了Condition的类就能实现条件装配。

比如下面 hello1 是否能实例化，取决于 Hello1Condition.matches方法，这个方法是根据能否load java.util.Date类。
这在SpringBoot中用来实现自动装配。


```java
package andy.com.springFramework.core.annotation;

import org.springframework.context.annotation.*;
import org.springframework.core.type.AnnotatedTypeMetadata;

/**
 * 条件实例化 Bean
 */
@Configuration
public class TestCondition {

    public static void main(String[] args) {

        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(TestCondition.class);

        /**
         * 可以load java.util.Date hello1被实例化
         */
        String hello1 = context.getBean("hello1", String.class);
        System.out.println(hello1);


        /**
         *  hello2不被实例化
         */
        String hello2 = context.getBean("hello2", String.class);
        System.out.println(hello2);
    }


    @Bean("hello1")
    @Conditional(Hello1Condition.class) //可以load java.util.Date hello1被实例化
    public String hello() {
        return "hello world1";
    }

    @Bean("hello2")
    @Conditional(Hello2Condition.class) //Hello2Condition.matches 返回 false 不会被实例化
    public String hello2() {
        return "hello world2";
    }

    public static class Hello1Condition implements Condition {

        /**
         * 如果环境中有 java.util.Date 类就返回true
         *
         * @param context
         * @param metadata
         * @return
         */
        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            try {
                context.getClassLoader().loadClass("java.util.Date");
                return true;
            } catch (Exception e) {
                return false;
            }
        }
    }

    /**
     * 返回 false
     */
    public static class Hello2Condition implements Condition {

        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
            return false;
        }
    }
}

```

springboot 自动配置也就是上面的原理
比如 classpath中有 "org.springframework.jdbc.core.JdbcTemplate" 类才会去初始化一个 JdbcTemplate

```java
public class JdbcTemplateCondition implements Condition {
      @Override
      public boolean matches(ConditionContext context,
                             AnnotatedTypeMetadata metadata) {
        try {
          context.getClassLoader().loadClass(
                 "org.springframework.jdbc.core.JdbcTemplate");
          return true;
        } catch (Exception e) {
          return false;
        } 
    }
}

```


springboot还提供了很多Condition的注解
```java
@ConditionalOnBean           配置了某个特定的bean
@ConditionalOnMissingBean    没有某个特定的bean
@ConditionalOnClass          classpath有某个指定的类
@ConditionalOnMissingClass   classpath没有某个自定的类
@ConditionalOnExpression     给定的Spring Expression Language 表达式为true
...
...
```