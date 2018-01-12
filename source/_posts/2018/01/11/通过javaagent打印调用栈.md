---
title: 通过javaagent打印调用栈
date: 2018-01-11 09:13:41
categories: ['javaagent']
tags: ['javaagent', '调用栈', '运行时间']
---

在平常工作中有时候需要查看方法的调用时间，或者需要知道某个业务的调用逻辑，需要些大量侵入式代码来完成。现在可以使用javaagent在main方法前执行，然后加载的类，通过字节码技术，在类中加入需要的代码。

<!-- more -->
### 构建agent项目
1、 新建maven jar项目
pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.whh</groupId>
    <artifactId>javaagentdemo</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>javaagentdemo</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!--
        <dependency>
            <groupId>org.apache.bcel</groupId>
            <artifactId>bcel</artifactId>
            <version>6.2</version>
        </dependency>
      -->
      <!-- 字节码增强 -->
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.21.0-GA</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
          <!-- 打包依赖jar -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <!-- 用这个maven打包插件 -->
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>

        </plugins>
    </build>
</project>
```
2、 新建java文件、新增premain方法如下：
```java
public class PreMain {
    public static void premain(String args, Instrumentation instrumentation){
        System.out.println("premain");
    }
}
```
3、resource下新建META-INF/MANIFEST.MF
```MF
Manifest-Version: 1.0
Premain-Class: com.whh.PreMain
Can-Redefine-Classes: true

```
* 注意的是最后是一个空行，没有会报错。

4、 打包生成jar
5、 随便写一个Main启动测试、在启动时添加VM参数：-javaagent:javaagentdemo-1.0-SNAPSHOT.jar。会发现PreMain中premain会被执行。

### 字节码修改
我们需要对加载的类做字节码修改，所以需要用到Instrumentation。
新建类TransformerDemo.java
```java

import javassist.*;

import java.io.IOException;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.ArrayList;
import java.util.List;

/**
 * TestTransformer
 * Created by whhxz on 2018/1/9.
 */
public class TransformerDemo implements ClassFileTransformer {

    private static ThreadLocal<List<MethodStackInfo>> inMethodStack = ThreadLocal.withInitial(ArrayList::new);

    /**
     * 方法调用前调用，记录进入方法时间
     * @param className
     * @param method
     */
    public static void startMethod(String className, String method) {
        long now = System.currentTimeMillis();
        MethodStackInfo methodStackInfo = new MethodStackInfo(className + "." + method);
        methodStackInfo.setStartTime(now);
        List<MethodStackInfo> inStack = inMethodStack.get();
        if (inStack.size() == 0) {
            methodStackInfo.setDepth(0);
        } else {
            for (int i = inStack.size() - 1; i >= 0; i--) {
                MethodStackInfo lastInStack = inStack.get(i);
                if (lastInStack.getEndTime() == 0) {
                    methodStackInfo.setDepth(lastInStack.getDepth() + 1);
                    break;
                }
            }
        }
        inStack.add(methodStackInfo);
    }

    /**
     * 出方法时调用，记录出方法时间
     * @param className
     * @param method
     */
    public static void endMethod(String className, String method) {
        long now = System.currentTimeMillis();
        List<MethodStackInfo> inStack = inMethodStack.get();
        for (int i = inStack.size() - 1; i >= 0; i--) {
            MethodStackInfo methodStackInfo = inStack.get(i);
            if (methodStackInfo. getEndTime() == 0) {
                methodStackInfo.setEndTime(now);
                break;
            }
        }
        //最外层已经出栈，打印相关信息
        if (inStack.get(0).getName().equals(className + "." + method)) {
            for (MethodStackInfo methodStackInfo : inStack) {
                StringBuilder sb = new StringBuilder();
                sb.append("|");
                for (int i = 0; i < methodStackInfo.getDepth(); i++) {
                    sb.append("  |");
                }
                sb.append("__").append(methodStackInfo.getName())
                        .append(" :")
                        .append(methodStackInfo.getEndTime() - methodStackInfo.getStartTime())
                        .append(" ms");
                System.out.println(sb.toString());
            }
            inMethodStack.remove();
        }
    }

    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        className = className.replaceAll("/", ".");
        //过滤需要处理的类
        if (!className.contains("com.whh.Main")
                || className.contains("$$")) {
            return null;
        }
        try {
            //获取加载的字节码
            ClassPool classPool = ClassPool.getDefault();
            classPool.insertClassPath(new LoaderClassPath(loader));
            CtClass ctClass = classPool.get(className);
            //修改方法字节码，新增插入的代码
            CtMethod[] declaredMethods = ctClass.getDeclaredMethods();
            for (CtMethod declaredMethod : declaredMethods) {
                String methodName = declaredMethod.getName();
                if (declaredMethod.isEmpty()) continue;
                declaredMethod.insertBefore("com.whh.transformer.TestTransformer.startMethod(\"" + className + "\", \"" + methodName + "\");");
                declaredMethod.insertAfter("com.whh.transformer.TestTransformer.endMethod(\"" + className + "\", \"" + methodName + "\");", true);
            }
            /*for (CtMethod declaredMethod : declaredMethods) {
                String oldName = declaredMethod.getName();
                declaredMethod.setName(oldName + "$old");
                CtMethod newMethod = CtNewMethod.copy(declaredMethod, oldName, ctClass, null);
                StringBuilder sb = new StringBuilder();
                sb.append("{")
                        .append("\nlong startTime = System.currentTimeMillis();\n")
                        .append(oldName).append("$old($$);\n")
                        .append("\nlong endTime = System.currentTimeMillis();\n")
                        .append("\nSystem.out.println(\"~~~~~~~~this method ")
                        .append(oldName)
                        .append(" cost:\" +(endTime - startTime) +\"ms.\");")
                        .append("}");
                newMethod.setBody(sb.toString());
                ctClass.addMethod(newMethod);
            }*/

            return ctClass.toBytecode();
        } catch (NotFoundException | CannotCompileException | IOException e) {
            System.out.println("~~~~~~~~~~" + className);
            e.printStackTrace();
        }
        return new byte[0];
    }
}
//MethodStackInfo存储方法调用信息
public class MethodStackInfo {
    private String name;
    private long startTime;
    private long endTime;
    private int depth;

    public MethodStackInfo() {
    }

    public MethodStackInfo(String name) {
        this.name = name;
    }

    public MethodStackInfo(String name, long startTime, long endTime, int depth) {
        this.name = name;
        this.startTime = startTime;
        this.endTime = endTime;
        this.depth = depth;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public long getStartTime() {
        return startTime;
    }

    public void setStartTime(long startTime) {
        this.startTime = startTime;
    }

    public long getEndTime() {
        return endTime;
    }

    public void setEndTime(long endTime) {
        this.endTime = endTime;
    }

    public int getDepth() {
        return depth;
    }

    public void setDepth(int depth) {
        this.depth = depth;
    }
}
```
修改PreMain方法
```java
public class PreMain {
    public static void premain(String args, Instrumentation instrumentation){
        System.out.println("premain");
        //新增
        instrumentation.addTransformer(new TransformerDemo());
    }
}
```
项目打包后通过之前方法测试即可。

### 遇到的问题
1、通过Tomcat启动时获取不到类的字节码
解决：因为Tomcat启动时使用多个类加载器作为系统类加载器。这时需要使用insertClassPath来解决

2、部分方法无法修改
解决：过滤抽象方法

### 后续问题

1、如果代码中使用循环，最好是能识别出来或者在后续打印过程中去掉
2、如果有死循环需要特别处理

这个例子是在main方法启动前，还有get在main方法启动后agentmain。
不想在启动时加入VM参数可以参考lombok、stagemonitor的相关实现。

参考：
* [Javassist 使用指南（一）](https://www.jianshu.com/p/43424242846b)
* [Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)
* [利用 Javassist 进行面向方面的更改](https://www.ibm.com/developerworks/cn/java/j-dyn0302/index.html?ca=drs-)
* [Java 5 特性 Instrumentation 实践](https://www.ibm.com/developerworks/cn/java/j-lo-instrumentation/)
