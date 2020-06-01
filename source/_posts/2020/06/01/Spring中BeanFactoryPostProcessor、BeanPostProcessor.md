---
title: Spring中BeanFactoryPostProcessor、BeanPostProcessor
date: 2020-06-01 10:27:36
categories: ['Spring']
tags: ['BeanFactoryPostProcessor', 'BeanPostProcessor']
---

### BeanFactoryPostProcessor
实现改接口，可以在Bean创建前，修改Bean定义的对象（元数据），也就是定义Bean的相关信息。但是需要注意的是，不可以在该实现里面触发Bean的实例化操作。可能会导致其他意想不到的问题。文档中有注释，如果需要与构建的Bean有交互的话使用**BeanPostProcessor**

### BeanPostProcessor
该接口有两个方法。一个为执行Bean的init方法之前，一个是之后，在该过程中可以对该Bean进行修改操作。
<!-- more -->
如果现在某个Bean的处理有上面两个实现，整个流程如下：
> 1、Spring读取配置文件，构建BeanFactoryPostProcessor，构建Bean的定义（元数据）
2、执行BeanFactoryPostProcessor中实现方法
3、创建Bean
4、执行BeanPostProcessor中postProcessBeforeInitialization
5、执行Bean中init方法
6、执行BeanPostProcessor中postProcessAfterInitialization