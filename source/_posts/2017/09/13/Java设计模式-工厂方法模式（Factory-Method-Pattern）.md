---
title: Java设计模式-工厂方法模式（Factory Method Pattern）
date: 2017-09-13 09:37:34
categories: ['设计模式']
tags: ['设计模式', '工厂方法', 'Factory Method Pattern']
---

### 工厂方法模式
> 定义一个用于创建对象的接口，让子类决定将哪一个类实例化。工厂方法模式让一个类的实例化延迟到其子类。工厂方法模式提供一个抽象接口来声明抽象工厂方法，而由其子类来具体实现工厂方法，创建具体的产品对象。

如图工厂方法：
![](http://otxnth5wx.bkt.clouddn.com/20170913%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-13%E4%B8%8A%E5%8D%8810.14.46.png)
* Logger：抽象日志类
* FileLogger：文件日志
* ConsoleLogger：控制台日志
* LoggerFactory：工厂类接口
* FileLoggerFactory：具体的文件工厂类
* ConsoleLoggerFactory：具体的控制台日志工厂类

Java代码如下：
```java
interface LoggerFactory {
    Logger createLogger();
}

class FileLoggerFactory implements LoggerFactory {

    @Override
    public Logger createLogger() {
        FileLogger fileLogger = new FileLogger();
        //do other something
        System.out.println("创建成功：" + fileLogger.getClass().getSimpleName());
        return fileLogger;
    }
}

class ConsoleLoggerFactory implements LoggerFactory {
    @Override
    public Logger createLogger() {
        ConsoleLogger consoleLogger = new ConsoleLogger();
        //do other something
        System.out.println("创建成功：" + consoleLogger.getClass().getSimpleName());
        return consoleLogger;
    }
}

interface Logger {
    void writeLog();
}

class FileLogger implements Logger {
    @Override
    public void writeLog() {
        System.out.println("文件日志输出");
    }
}

class ConsoleLogger implements Logger {

    @Override
    public void writeLog() {
        System.out.println("控制台日志输出");
    }
}

public class Main {
    public static void main(String[] args) {
        Logger fileLogger = new FileLoggerFactory().createLogger();
        Logger consoleLogger = new ConsoleLoggerFactory().createLogger();
        fileLogger.writeLog();
        consoleLogger.writeLog();
    }
}
```
### 工厂模式优点
1. 在工厂设计模式中，工厂方法用于创建所需要的对象。调用方不需要关心创建细节，只需要找到对应的工厂创建对应的对象。
2. 在需求变更时，需要新增一个产品。只需要新建相应的工厂类就可以，无需变动原有代码，使可扩展性变强，符合**开闭原则**。
3. 方便维护，对象一致性。在需要修改创建的对象，或者在创建对象前后有变更相关操作，可以直接修改Factory中创建对象的前后操作。如果每个地方都自己用new创建，修改地方太多，存在风险。

### 工厂模式缺点
1. 每次新增一个产品，都需要写个新的工厂类。
2. 考虑了可扩展性，引入了抽象层次，加大系统理解难度

### 工厂模式总结
在Spring中大量使用了工厂设计模式（可参考FacoryBean），由Spring这个大工厂创建用户所需要的对象，组装对象，用户需要时，由Spring工厂创建对象提供给用户使用。减少了用户自己操作出错概率，各个对象之间也可以解耦。
在JDK中还有
```
java.lang.Proxy#newProxyInstance()
java.lang.Class#newInstance()
java.lang.reflect.Constructor#newInstance()
```
