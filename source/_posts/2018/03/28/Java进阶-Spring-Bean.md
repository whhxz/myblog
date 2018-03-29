---
title: Java进阶-Spring Bean
date: 2018-03-28 18:14:36
categories: ['Java进阶']
tags: ['Spring', 'Bean']
---

Spring作为JavaWeb流行框架，其核心之一就是Bean的管理。其中有Bean的创建、管理、加载。
添加SpringBean依赖，启动Spring。
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.2.7.RELEASE</version>
</dependency>
```
添加Spring配置文件application.xml
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <context:annotation-config />
</beans>
```
<!-- more -->
启动Spring容器
```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:application.xml");
        for (String beanName : applicationContext.getBeanDefinitionNames()) {
            System.out.println(beanName);
        }
    }
}
```
最简单的Spring容器启动就是这样。

### SpringBean创建
进入ClassPathXmlApplicationContext分析Spring加载Bean的过程。
默认加载流程图如下（简化版）：
![]()

基本就是先创建BeanFacotry（用于后续构建Bean）---> 读取配置 ---> 封装需要创建的Bean
在这过程中有不少前置后置等相关处理。
* 在处理不同的配置时，由不同的Handler处理，可以通过spring.handlers来查看。

上述构造好需要组装的Bean后（Bean还未创建），之后通过BeanFactory创建需要的Bean。
默认加载流程图如下（单例简化版）：
![]()

在之前创建的BeanFactory实际上创建的默认类为`DefaultListableBeanFactory`
需要注意的是，在创建Bean的过程中，如果该Bean有其他依赖，先创建Bean后对其依赖的属性进行赋值，如果赋值的Bean不存在，会先创建该Bean，直到创建完成。

* 如果是通过构造方法注入，两个Bean互相依赖，而且都是通过构造方法创建，那么就会出现死循环导致Spring启动异常。

### Bean作用域
Spring Bean创建有的对象有5类作用域：singleton、prototype、request、session、global session

singleton：单例模式，就是在Spring容器中只存在一个改对象，生命周期随着Spring而走，当Spring容器关闭后随之销毁。
prototype：每次使用都会创建一个新的对象。
request：一般随着Web使用，一次request请求创建一次
session：随着Web使用，在一次seesion过程中创建一次
global session：全局session一次，一般用于Portlet，使用较少

singleton、prototype创建可以查看代码`org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean`，这里有判断创建的Bean的作用域，然后通过不同的作用域创建对象。

对于request、session是通过不同的`org.springframework.beans.factory.config.Scope`所创建。

* 如果需要一个singleton注入的是prototype属性，那么需要使用`@Scope proxyMode`，这样生成的bean就是有Spring生成的代理类，在调用时会生成不同的Bean，request、seesion同理，一有多。

### Bean生命周期
1、创建实例
2、设置属性
3、调用初始化方法
4、放入容器，应用可以通过容器获取Bean
5、容器销毁，调用Bean销毁

在使用过程中，创建的Bean实现不同的接口，创建时间不一样。

* 在使用Spring时，使用xml配置时，默认是通过byName来确定对象，也就是配置的Bean id，通过注解扫描时默认是通过byType。