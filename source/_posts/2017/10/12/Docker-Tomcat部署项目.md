---
title: Docker Tomcat部署项目
date: 2017-10-12 14:36:48
categories: ['Docker']
tags: ['Docker']
---
### 下载配置相关软件
配置自定义Docker镜像，镜像系统为了省事直接用CentOS，直接在使用命令下载：
```sh
docker install centos
```
需要下载JRE和Tomcat，Jre在Oracle官网下载Jre8和Tomcat官网下载。

Jre下载解压后不需要做相应改动。
Tomcat在解压后，删除 **`webroot/ROOT`** 下所有文件。修改 **`conf/server.xml`**，在Host节点下添加 ** `&lt;Context docBase="/home/webdata/webroot/manage.war" path="" reloadable="true" /&gt;` ** 。`manage.war`表示这次配置的项目。<!-- more -->
### 配置Dockerfile
```file
# 使用centos镜像
FROM centos
MAINTAINER whh "xuzhuowhh@gmail.com"
# 创建jre目录
RUN mkdir /usr/local/jre
# 创建tomcat目录
RUN mkdir /usr/local/tomcat
# 和之前tomcat中docBase中配置的目录保持一致
RUN mkdir -p /home/webdata/webroot
# 添加本地jre目录到自定义镜像目录
ADD jre1.8.0_144 /usr/local/jre
# 添加本地tomcat目录到自定义镜像目录
ADD apache-tomcat-9.0.1 /usr/local/tomcat/
# 配置环境变量
ENV JRE_HOME /usr/local/jre
ENV PATH $PATH:$JRE_HOME/bin

# 暴露端口
EXPOSE 8080
# 镜像启动时启动tomcat
CMD ["/usr/local/tomcat/bin/catalina.sh", "run"]
```

配置完成后，构建自定义镜像
```sh
docker build -t whhxz/manage .
```
* 注：最后面有个点
### 启动项目
启动项目：
```sh
docker run -d -v /home/whh/Documents/docker/webroot/:/home/webdata/webroot -p 8080:8080 whhxz/manage
```
> -d：表示后台运行
> -v：表示挂载本地目录到容器目录中
> -p：表示映射本地端口和容器端口

启动完成后可以直接访问项目地址。
在最开始配置的时候，会出现各种各样的问题，如果容器启动不起来，可以通过命令进入容器里面
```sh
docker run -d -i -t -v /home/whh/Documents/docker/webroot/:/home/webdata/webroot -p 8080:8080 whhxz/manage /bin/bash
```
启动容器后，使用docker ps查看运行的容器，获取`CONTAINER ID`，通过命令 **docker attach `CONTAINER ID`** 进入容器，调试出现的问题，一步步改进配置。
* 注：在使用`CONTAINER ID`的时候，只使用前四位也可以。
### 后续需要改进
* 1、日志的收集以及查看
* 2、在实际使用过程中，最好是通过自定义脚本启动tomcat，在脚本中，清理tomcat缓存，避免出现问题。
