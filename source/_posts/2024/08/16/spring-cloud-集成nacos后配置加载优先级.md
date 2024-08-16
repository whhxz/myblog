---
title: spring cloud 集成nacos后配置加载优先级
date: 2024-08-16 09:18:44
categories:
tags: ['spring cloud', 'nacos']
---

项目中使用nacos统一管理配置，在本地开发时需要修改配置，发现启动时添加`-Dxx=xx`并不生效。
<!-- more -->
```java
class  org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration

//获取到nacos的`PropertySource`
doInitialize
//设置优先级
insertPropertySources
```
通过上述nacos里面`PropertySources`获取如下属性来判配置优先级

* spring.cloud.config.allowOverride (默认true)
* spring.cloud.config.overrideNone (默认false)
* spring.cloud.config.overrideSystemProperties (默认true)

| allowOverride | overrideNone | overrideSystemProperties | nacos优先级 |
| --- | --- | --- | --- |
| false | - | - | 最高 |
| true | false | true | 默认，最高 |
| - | true | - | 最低 |
| - | - | true | 高于`systemEnvironment` |
| - | - | false | 低于`systemEnvironment` |
| - | - | - | 最低 |
> 上述判断是从上往下判断，匹配就直接执行
> 如果存在`systemEnvironment`才会依据`overrideSystemProperties`判断相对`systemEnvironment`优先级
