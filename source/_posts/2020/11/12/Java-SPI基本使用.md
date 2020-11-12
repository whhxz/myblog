---
title: Java SPI基本使用
date: 2020-11-12 14:52:25
categories: ['Java基础']
tags: ['SPI']
---

之前看slf4j-api源码时，2.0版本中切换不同的日志，采用的就是SPI。通过定义接口，不同的日志框架实现该接口，对于使用方而言，通过JDK提供的方法找到实现的类并构建对象。接口不直接实现，又其他第三方实现该接口，支持热插拔。

> slf4j-api 1.*版本并不是用的这种方法，是通过自定义类**org.slf4j.impl.StaticLoggerBinder**，实现使用不同的日志。

<!-- more -->
SPI全称为**Service Provider Interface**。

1. 构建项目A，添加接口
```java
public interface SPIInterface {
    
    String read();
    
    void write();
}
```
2. 构建项目B，添加上面接口实现
```java
public class SPIDemo implements SPIInterface {
    public String read() {
        return "whhxz";
    }

    public String write(String name) {
        return name + "-";
    }
}
```
在项目B中，在**resource**下面添加目录**META-INF/services**，在该目录下添加文件**com.whhxz.demo.SPIInterface**（接口名称），文件内容为**com.whhxz.demo.SPIDemo**（实现类名）

3. 构建项目C，在项目C中添加B依赖
```java
public class Main {
    public static void main(String[] args) {
        ServiceLoader<SPIInterface> demo = ServiceLoader.load(SPIInterface.class);
        Iterator<SPIInterface> iterator = demo.iterator();
        while (iterator.hasNext()){
            SPIInterface spi = iterator.next();
            System.out.println(spi.read());
            System.out.println(spi.write("whh"));
        }
    }
}
```
在项目C中，通过**ServiceLoader.load**获取接口的实现。实现原理是通过扫描获取实现类，通过**newInstance**构建对象。

在使用SPI中，缺点是不是线程安全，且只能一次获取所有的接口，如果有大量实现时，可能性能较慢。