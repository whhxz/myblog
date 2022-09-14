---
title: spring-boot打包jar问题
date: 2022-09-14 17:09:53
categories: ['SpringBoot', 'Maven']
tags: ['SpringBoot', 'Maven', '工作问题', '依赖']
---

最近公司要求修改依赖为公司统一封装的SpringBoot，里面包含了注册中心等配置，方便统一管理。

开发SpringBoot项目一般默认继承**spring-boot-starter-parent**，这次要求默认继承公司内部的依赖。同时把以前打包的**war**改为**jar**，最后打包发现只是普通的jar打包编译，也没有依赖。
<!-- more -->
原本使用的*spring-boot-maven-plugin*如下
```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<fork>true</fork>
	</configuration>
</plugin>
```
解决办法如下，添加配置参数。
```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
		<fork>true</fork>
		<mainClass>com.rtmart.promotion.api.RTMartPromotionApiApplication</mainClass>
		<layout>ZIP</layout>
	</configuration>
	<executions>
		<execution>
			<goals>
				<goal>repackage</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```
* 经过测试**mainClass**和**layout**可以选择不要，一样可以生效

> 参考 https://www.cnblogs.com/acm-bingzi/p/mavenspringbootplugin.html