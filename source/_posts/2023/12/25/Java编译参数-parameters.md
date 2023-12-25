---
title: Java编译参数-parameters
date: 2023-12-25 15:59:34
categories:
tags: ['java']
---

有个项目最开始gradle，打算边写边学习，最近不知道为什么一直编译失败，一直没找到原因。索性又改为maven了，在改的时候springboot采用的是`3.1.5`，里面的spring版本是`6.0`。在改为maven时顺手改为`3.2.1`，之后调用接口一直失败。提示
```
exception Name for argument of type [java.lang.Integer] not specified, and parameter name information not available via reflection. Ensure that the compiler uses the
 '-parameters' flag.
```
其实在spring wiki里面有写，升级注意事项，之前没看见，后面才注意，因为`LocalVariableTableParameterNameDiscoverer`在spring6.1里面已经删除。
> [Upgrading-to-Spring-Framework-6.x](https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-6.x#parameter-name-retention)
> [Spring-Boot-3.2-Release-Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.2-Release-Notes)

以前`LocalVariableTableParameterNameDiscoverer`是会读取解析对象class文件后通过`LocalVariableTable`拿到参数名称。
<!-- more -->
解决办法是添加
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <parameters>true</parameters>
    </configuration>
</plugin>
```
> 如果使用的是`spring-boot-starter-parent`，然后添加了插件`spring-boot-maven-plugin`是没问题的，因为插件默认会添加`-parameters`

 ### 编译测试
 创建一个类：
 ```java
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class Demo {
    public static void main(String[] args) throws Exception {
        Method method = Demo.class.getMethod("demo", Integer.class);
        Parameter[] parameters = method.getParameters();
        for (Parameter parameter : parameters) {
            System.out.println(parameter.getName());
        }
    }

    public void demo(Integer id) {
        System.out.println(id);
    }
}
 ```
使用命令编译`javac Demo.class`然后运行`java Demo`输出得到 **arg0**
添加参数后命令`javac -parameters Demo.java`，然后运行得到 **id**
实际上通过`javap -v`反编译class后可以看到添加参数后会多出 **MethodParameters** 里面就有名称 **id**
如果通过添加参数`javac -g:vars`编译，反编译后可以看到里面有添加`LocalVariableTable`，之前应该是maven编译的默认存在，所以之前Spring能获取到参数名称