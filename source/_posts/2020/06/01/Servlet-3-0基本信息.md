---
title: Servlet 3.0基本信息
date: 2020-06-01 10:48:18
categories: ['Servelt']
tags: ['Servelt 3.0']
---

Servelt 3.0在2009年已经随着JavaEE 6推出。主要增加了异步处理、注解支持、模块化处理

### 异步支持
在之前Servelt在接收到请求后需要处理完毕后再做返回，线程一直在阻塞状态，现在可以接收数据后交由其他线程处理，线程接收线程本身返回容器。如果聊天中等待消息。
在**WebServlet**设置**asyncSupported**为true即可(默认为false)
<!-- more -->
### 模块化支持
支持在容器启动后可以动态在**ServletContext**中添加**Servelt**，在SpringBoot中内嵌Tomcat，在启动Tomcat后，通过动态添加**DispatcherServlet**到容器中。

参考：
* [Servlet 3.0 新特性详解](https://www.ibm.com/developerworks/cn/java/j-lo-servlet30/index.html)
* [JavaEE 6 Servlet 3.0 中的新特性](https://www.oracle.com/technetwork/cn/community/4-servlet-3-324302-zhs.pdf)
