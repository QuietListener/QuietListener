---
layout: post
title: springboot2搭建mockito 记录
date: 2022-4-15 14:32:00
categories:  测试
---

## 依赖

```java
  <dependency>
                <groupId>org.mockito</groupId>
                <artifactId>mockito-core</artifactId>
                <version>2.23.4</version>
            </dependency>


            <dependency>
                <groupId>org.powermock</groupId>
                <artifactId>powermock-module-junit4</artifactId>
                <version>2.0.2</version>
            </dependency>

            <dependency>
                <groupId>org.powermock</groupId>
                <artifactId>powermock-api-mockito2</artifactId>
                <version>2.0.2</version>
            </dependency>

            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>4.12</version>
            </dependency>
```



```java

@RunWith(PowerMockRunner.class)
@PowerMockRunnerDelegate(SpringJUnit4ClassRunner.class)
@PowerMockIgnore({"java.lang.management.*","javax.management.*","sun.security.*", "javax.net.*"})
@SpringBootTest(classes = SpringbootServiceUserTestApplication.class,webEnvironment= SpringBootTest.WebEnvironment.RANDOM_PORT)
@Commit
public class BaseTest {
    @LocalServerPort
    int randomServerPort;
}
```

```java

@Service
public class MockitoService {

    public static final String PREFIX = "MockitoService";

    @Autowired
    SubMockitoService subMockitoService;

    @Autowired
    SomeConfig someConfig;

    public String getSomeString(long any) {
        return PREFIX + any;
    }


    public String getSomeMore(long any) {
        String ret =   subMockitoService.getSomeInt(any);
        return ret;
    }


    public SomeConfig getSomeConfig(){
        return someConfig;
    }
}

```

```java

@Configuration
public class SomeConfig {



    private String config;

    @PostConstruct
    public void init(){
        config  = "111";
    }

    public String getConfig() {
        return config;
    }

    public void setConfig(String config) {
        this.config = config;
    }
}
```

```java

@Service
public class SubMockitoService {

    public static final String PREFIX = "SubMockitoService";

    public String getSomeInt(long any) {
        return PREFIX + any;
    }

}

```



### 测试类
```java

@PrepareForTest
public class TestMokito1 extends BaseTest {

    @SpyBean
    private MockitoService mockitoService;

    @SpyBean
    private SubMockitoService subMockitoService;

    @SpyBean
    private SomeConfig someConfig;

    @Before
    public void init() {
        MockitoAnnotations.initMocks(this);
    }



    @Test
    public void test1() {

        String retNormal = mockitoService.getSomeString(111l);
        assert retNormal.equals(MockitoService.PREFIX+111l);

        PowerMockito.when(subMockitoService.getSomeInt(111l)).thenReturn("111");
        String ret = mockitoService.getSomeMore(111l);
        assert ret.equals("111");


        PowerMockito.when(mockitoService.getSomeString(111l)).thenReturn("111");
        String ret1 = mockitoService.getSomeString(111l);
        assert ret1.equals("111");


        SomeConfig someConfig = mockitoService.getSomeConfig();
        assert someConfig != null ;
        assert someConfig.getConfig() != null ;

        PowerMockito.when(someConfig.getConfig()).thenReturn("ok");
        String configRet = someConfig.getConfig();
        assert "ok".equals(configRet);
    }


}

```



## @mock @spay @mockBean @spyBean的区别
1. @mock和@spy不受spring管理 
2. @mockBean和@spyBean受spring管理，会自动替换@Autowired的注入
3. 一个方法没有mock spy会调用真实方法返回真实值，   mock默认不执行有返回值，直接返回null。
4. 