---
title: snake-agent初步搭建
date: 2018-12-27 21:16:21
categories: ['snake']
tags: ['agent', 'classLoader']
---

准备从0搭建一个框架，设计调用链，写下构建过程。

### agent基础使用
在JDK6的时候，更新了javaagent，在启动前会执行jar中配置的premain。在启动项目前在代码中对加载的class进行字节码修改，插入特定代码，如日志输入输出，达到项目跟踪的目的。
<!-- more -->
### classLoader加载lib
在agent中通过使用``System.getProperty("java.class.path")``获取启动的classPath，通过解析值可以获取到启动agentJar路径等相关值，从而定位到目录地址，通过获取到的目录地址得到获取项目的配置文件以及相关依赖的jar。

加载解析配置文件，通过自定义classLoader加载依赖的jar。达到agent的启动项目的依赖分离而不会产生jar的冲突。

### 加载指定启动类
通过之前设置的classLoader加载需要加载的class，启动项目。在classLoader中加载的``DefaultAgent``中输出日志。

### 小结
1、通过javaagent添加启动参数
2、通过classLoader加载指定class
3、class构建对象
4、启动项目

看项目tar0.0.1