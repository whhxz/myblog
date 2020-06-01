---
title: Spring中BeanFactory、FactoryBean、ObjectFactory
date: 2020-06-01 10:09:28
categories: ['Spring']
tags: ['BeanFactory', 'FactoryBean', 'ObjectFactory']
---


### BeanFactory
是Spring IOC容器定义的跟接口，用于获取Bean。

### FactoryBean
是一个Bean，只不过是一个工厂Bean，在BeanFactory需要获取Bean的时候，通过FactoryBean来生成对象返回，如果是单例，缓存生成的对象。如果需要获取的是FactoryBean而不是FactoryBean生成Bean时，需要在Bean的名称上面加一个**&**

<!-- more -->
### ObjectFactory
也是用于生成Bean，只不过生成Bean的时间由自己把握
在BeanFactory.getBean的时候，是通过FactoryBean生成Bean返回
如果需要生成的Bean又ObjectFactory构建，那么需要先获取ObjectFactory之后再调用ObjectFactory.getObject

ObjectFactory和FactoryBean区部在生成Bean的时机不同