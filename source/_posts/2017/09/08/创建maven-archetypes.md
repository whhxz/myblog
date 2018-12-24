---
title: 创建maven archetypes
date: 2017-09-08 09:48:25
categories: ['maven']
tags: ['maven', '脚手架', '快速搭建项目', 'archetypes']
---

## 建立maven archetypes，后期快速搭建项目

#### 创建一个空的maven项目
修改当前maven项目pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.whh.maven.archetypes</groupId>
  <artifactId>quick-ssm-webapp</artifactId>
  <packaging>jar</packaging>
  <version>1.0.0</version>
  <name>quick-ssm-webapp Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <build>
    <finalName>quick-ssm-webapp</finalName>
    <pluginManagement>
      <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-archetype-plugin</artifactId>
            <version>2.4</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```
<!-- more -->
#### 配置相关文件
目录结构如下
![](http://image.whhxz.smallstool.cn/20170908%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-08%E4%B8%8A%E5%8D%8810.13.35.png)
目录**archetype-resources**下配置需要生成的项目，在使用maven生成项目时，目录结构和archetype-resources下目录一致。
目录**META-INF/maven**下*archetype-metadata.xml*设置生成规则
如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<archetype-descriptor name="sample">
    <fileSets>
      <!-- package为true表示通过groupId自动生成包 -->
      <!-- 如下配置表示archetype-resources下文件有效文件 -->
        <fileSet filtered="true" packaged="true">
            <directory>src/main/java</directory>
            <includes>
                <include>**/*</include>
            </includes>
        </fileSet>
        <fileSet filtered="true">
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.*</include>
            </includes>
        </fileSet>
        <fileSet filtered="true">
            <directory>src/main/webapp</directory>
            <includes>
                <include>**/*.**</include>
            </includes>
        </fileSet>
        <fileSet filtered="true">
            <directory>src/main/webapp/static</directory>
            <includes>
                <include>**/*.**</include>
            </includes>
        </fileSet>
        <fileSet filtered="true" packaged="true">
            <directory>src/test/java</directory>
            <includes>
                <include>**/*.*</include>
            </includes>
        </fileSet>
        <fileSet filtered="true">
            <directory>src/test/resources</directory>
            <includes>
                <include>**/*.*</include>
            </includes>
        </fileSet>
    </fileSets>
</archetype-descriptor>
```
* 注意事项：
> 在archetype-resources文件夹java代码中，package和import不要直写当前路径，可以使用${package}，在生成项目后，会自动替换，以免代码生成项目后，路径出错。
> 在pom.xml 或者其他配置文件，或其他java代码中有路径的，比如说spring扫描路径，可以配置为${groupId}，在创建项目时填写的groupId会被替换。同理还有${groupId}，其他相关配置可参考maven官方文档。

#### 使用方式
1. 使用命令 *mvn install* 把当前项目打包到本地maven仓库中，如果都要上传到服务器，可以配置好服务器路径，账号密码，使用 *mvn deploy* 上传文件。
2. 创建maven项目,如图![](http://image.whhxz.smallstool.cn/20170908QQ20170908-102803.png)，配置archetype相关参数。后续步骤和普通创建maven项目一致。
3. 导入创建的maven项目，配置数据库，启动项目即可。

该项目地址 ** [quick-ssm-webapp](https://github.com/whhxz/quick-ssm-webapp) **
