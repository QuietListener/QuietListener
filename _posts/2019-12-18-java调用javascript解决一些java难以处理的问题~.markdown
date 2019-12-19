---
layout: post
title: java调用javascript解决一些java难以处理的问题~
date: 2019-12-18 14:32:00
categories:  java javascript
---

# why
java作为静态语言，有的问题处理起来非常痛苦，例如一个非常复杂的并且异构的json字符串中去提取需要的数据就很痛苦。但是javascript天生就是处理json数据能手。这个时候使用java调用js脚本来处理这类数据，js处理完再将数据返回给java，就能发挥他们各自的优势。

# how
下面的例子使用

1. **java调用js的方法**  
2. **使用js实现javascirpt的接口**  


## 1. java 代码

```java
package andy.com.jsengine;

import javax.script.Invocable;
import javax.script.ScriptEngine;
import java.io.File;
import java.io.Reader;
import java.nio.charset.Charset;
import java.nio.file.Files;
import java.nio.file.Paths;


//get output of script
@SuppressWarnings("restriction")
public class Example5CallJsFunc {


    public static void main(String[] args) throws Exception {
        testCallJsFunc();
        testImplInterface();
    }

    //java调用js方法
    public static void testCallJsFunc() throws Exception {
        ScriptEngine engine = JSEngineUtils.getJsEngineInstance();

        File jsFile = JSEngineUtils.getFiles("example4_calljsfunc.js");
        Reader reader = Files.newBufferedReader(Paths.get(jsFile.getAbsolutePath()), Charset.forName("UTF8"));
        engine.eval(reader);
        Invocable inv = (Invocable) engine;
        Object caculator = engine.get("caculator");

        int x = 1;
        int y = 2;
        Object result = inv.invokeMethod(caculator, "add", x, y);
        System.out.println(result);
    }

    //js实现java的interface
    public static void testImplInterface() throws Exception {
        ScriptEngine engine = JSEngineUtils.getJsEngineInstance();

        File jsFile = JSEngineUtils.getFiles("example4_impl_interface.js");
        Reader reader = Files.newBufferedReader(Paths.get(jsFile.getAbsolutePath()), Charset.forName("UTF8"));
        engine.eval(reader);
        Invocable inv = (Invocable) engine;
        Math math = inv.getInterface(Math.class);

        int x = 3;
        int y = 2;
        double result = math.add(x, y);
        System.out.println(result);
    }

}
```



## 2. javascript代码

```javascript
//example4_calljsfunc.js
base = 10;

var caculator = new Object();

caculator.add = function (n1, n2) 
{
	return base + n1 + n2;
}

```


```javascript
//example4_impl_interface.js
function add (n1, n2) 
{
	return n1 + n2;
}
```



# 参考
1. [Scripting_in_Java](http://www.java2s.com/Tutorials/Java/Scripting_in_Java/index.htm)