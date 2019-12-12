---
layout: post
title: spring 使用profile 初始化Bean的一个坑
date: 2019-12-12 14:32:00
categories:  spring java
---

一个在项目中，使用 spring的@profile来控制配置文件，在使用-Dspring.profiles.active=test 明明指定了的profile为test的配置但是spring就是不初始化profile为test的变量。最后发现是一个spring的坑。
测试代码如下
```java
package andy.com.springFramework.bugs;

import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
public class ProfileBug {
    
    @Bean("name")
    @Profile("test")
    public String nameTest() {
        return "name_test";
    }

    //默认数据 没有放在最后定义
    @Bean("name")
    public String name() {
        return "name";
    }

    @Bean("name")
    @Profile("prod")
    public String nameProd() {
        return "name_prod";
    }

    public String testProfile(String profile) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(ProfileBug.class);
        context.getEnvironment().setActiveProfiles(profile);
        context.refresh();

        String name = (String) context.getBean("name");
        System.out.println(String.format("%s:%s", profile, name));
        return name;
    }

    @Test
    public void test1() {
        assert "name_test".equals(testProfile("test")); // passed return "name_test"
        assert "name".equals(testProfile("default")); // passed return default value "name"
        assert "name_prod".equals(testProfile("prod")); //failed return "name"
    }
}

```

名字为name的bean是一个String，定义了profile为test， prod， 外加一个default的值。按理说test1中的三个assert都应该通过，但是 第三个fail掉了。我们定义名字为name的bean的顺序是 把 默认值放在的中间，如果把默认的放在最后就能通过。如下

```java
package andy.com.springFramework.bugs;

import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;

@Configuration
public class ProfileBug {

    @Bean("name")
    @Profile("test")
    public String nameTest() {
        return "name_test";
    }

    @Bean("name")
    @Profile("prod")
    public String nameProd() {
        return "name_prod";
    }

    //默认数据 放在最后
    @Bean("name")
    public String name() {
        return "name";
    }



    public String testProfile(String profile) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(ProfileBug.class);
        context.getEnvironment().setActiveProfiles(profile);
        context.refresh();

        String name = (String) context.getBean("name");
        System.out.println(String.format("%s:%s", profile, name));
        return name;
    }

    @Test
    public void test1() {
        assert "name_test".equals(testProfile("test")); // passed return "name_test"
        assert "name".equals(testProfile("default")); // passed return default value "name"
        assert "name_prod".equals(testProfile("prod")); //passed return "name_prod"
    }
}

```


**注意这个坑，尽量将不同profile的bean放在不同的类中来定义，或者放在最后。**