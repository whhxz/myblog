---
title: quarkus构建demo
date: 2023-09-20 08:48:28
categories: ['quarkus']
tags: ['quarkus']
---

Quarkus可以很方便通过GraalVm编译本地文件后直接执行，编译后启动非常快。
本次通过构建Quarkus项目，然后插件通过docker编译后生成本地文件。

<!-- more -->

### 构建项目
可以通过官网教程创建，也可以通过idea直接创建Quarkus，本次为了方便快速体验，通过idea直接创建项目时选择Quarkus。
创建后可以在idea中看到项目下插件quarkus。
> 本次创建项目使用的版本为3.3.2

### 项目启动
可以通过maven命令`mvn compile quarkus:dev`
启动后默认端口8080，直接访问 http://localhost:8080 可以看到启动项目的相关信息。
访问示例接口 http://localhost:8080/hello 可以看到项目中示例接口返回值。

通过命令启动后默认调试端口为5005，可以启动idea的远程调试debug项目。

也可以通过idea直接debug启动。如果idea没有自动创建，可以在idea里面创建`quarkus`启动项目。

### 打包
通过下面命令打包jar
```shell
mvn clean package "-DskipTests" -U "-Dquarkus.package.type=uber-jar"
```
启动jar
```shell
java -jar .\target\quarkus-demo02-1.0-SNAPSHOT-runner.jar
```

打包二进制文件，需要Docker支持
```shell
mvn clean package -U "-DskipTests" "-Dnative" "-Dquarkus.native.container-build=true" "-Dquarkus.native.builder-image=quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-17"
```
之前创建项目使用的是`jdk17`，所以会通过docker拉取`quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-17`镜像，通过镜像对项目进行打包。不指定镜像也会默认使用该镜像。
打包逻辑是，线通过本地构建需要打包的jar和依赖，再启动docker容器，挂载需要的文件。再容器内通过命令编译为本地文件。

启动编译后文件，需要在wsl里面执行，编译后是linux可执行二进制文件。可以发现启动速度飞快。

参考：
> [ quarkus实战之二：应用的创建、构建、部署](https://www.cnblogs.com/bolingcavalry/p/17567289.html)