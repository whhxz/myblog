---
title: snake-agent拦截及日志输出
date: 2019-01-04 10:45:51
categories: ['snake']
tags: ['snake', 'classLoader']
---

### Spring请求拦截
在使用Spring框架时，可以知道所有Spring入口都是从``org.springframework.web.servlet.FrameworkServlet.processRequest``中进入，可以通过对该方法进行字节码增强，达到拦截的目的。
使用``javassist``对``processRequest``方法的前后分别插入操作代码。通过``javassist``提供获取当前对象出入参数等，做一些相关的操作。
<!-- more -->
#### 修改注册
在创建拦截操作对象时，为了保证在代码插入后生效，创建一个静态的注册类``InterceptorRegistry``，在创建代码时，生成一个唯一的id，通过``javassist``修改字节码时，带入该ID，就可以准确找到对应的修改类。

### 日志输出
在agent中，日志依赖需要使用自定义的类加载器加载，使agent依赖的jar和项目的jar进行隔离。这样就会出现，父类加载器（项目类加载器）需要调用子类加载器（自定义类加载器）加载的对象。

自定义日志类关系如下：
![](/images/old/20190104屏幕快照2019-01-04上午11.01.44.png)
其中接口``Logger``、``LoggerBinder``类``LogManager``由父类加载器加载，其实现类``Log4j2Logger``、``Log4j2Binder``由子类加载器加载。
在使用的时候，如果初始化LogManner传入Log4j2Logger达到类加载分离。因为在自定义类加载器中，只加载部分类，其他的类由父类加载。
方法的调用，通过接口进行连接，具体实现由子类加载器处理。

代码详情请看 v0.0.2