---
title: docker部署项目
date: 2018-04-23 10:01:25
categories: ['docker']
tags: ['部署', 'docker', 'tomcat']
---

在部署项目时，每次需要配置不同的环境，如tomcat，jdk等等，过于麻烦，如果直接使用docker做好镜像后，启动docker容器即可方便启动项目。

### docker配置tomcat
下载jdk、tomcat解压放到本地。
编写Dockerfile。
<!-- more -->
```Dockerfile
# 使用centos作为基础镜像
FROM centos
# 复制本地jkd、tomcat到镜像目录
COPY ./jdk8 /usr/local/share/jdk8
COPY ./tomcat9 /usr/local/share/tomcat9
# 配置运行时环境变量
ENV JAVA_HOME /usr/local/share/jdk8
ENV PAHT $JAVA_HOME/bin:$PATH
# 声明端口
EXPOSE 8080
# 启动tomcat，同时使用tail-f是因为了避免 docker自动执行完shell后直接停止
ENTRYPOINT /usr/local/share/tomcat9/bin/startup.sh && tail -F /usr/local/share/tomcat9/logs/catalina.out
```
上述就简单的配置生成镜像
命令`docker build -t tomcatweb --rm=true .`生成镜像。
> -t：用于生成镜像名
> --rm：指定在生成镜像过程中删除中间产生的临时容器
> .：表示当前目录下的Dockerfile

之后通过`docker images`查看生成的镜像。

启动镜像`docker run -d -p 8080:8080 tomcatweb`，之后会生成一个运行中的容器，通过命令`docker ps`可查看容器。
> -d：生成的容器后台运行
> -p：表示本机8080端口与容器8080端口进行绑定，方便访问

容器其他操作：
> docker ps -a：查看所有容器，包括停止的容器
> docker stop [CONTAINER ID]：表示停止容器
> docker start [CONTAINER ID]：启动容器
> docker kill [CONTAINER ID]：直接杀死容器
> docker rm [CONTAINER ID]：删除容器
> docker rm $(docker ps -a -q)：删除本机所有容器

启动后，通过`curl localhost:8080`可查看到tomcat已经启动完成，也可以直接浏览器访问

### docker-compose集成启动项目
现在公司的项目一般是直接运行在tomcat根目录下，一个项目一个tomcat。
所以需要删除tomcat解压后webapps下所有文件。
重新运行之前命令生成新的镜像。
编写docker-compost.yml文件
```yml
# 这里不能为1，会出错
version: '2'
# 启动的服务
services:
  web:
    # 使用镜像
    image: tomcatweb
    #使用的端口
    ports:
      - "8080:8080"
    # 挂载目录，在使用本地目录挂载到容器中时，本地目录需要使用绝对路径，相对路径可能会报错
    volumes:
      # 用于存放配置文件，因为在项目启动时，配置文件是外放，并不放在war包中，
      - /home/xxx/Documents/dockerstd/webroot/:/home/xxx/webroot
      # 日志文件，用于容器销毁后，日志文件还是保留在本地，因为容器中的数据会随着容器的销毁而销毁
      - /home/xxx/Documents/dockerstd/logs/:/usr/local/share/tomcat9/logs
      # 用于项目在tomcat根目录启动
      - /home/xxx/Documents/dockerstd/ROOT.xml:/usr/local/share/tomcat9/conf/Catalina/localhost/ROOT.xml
```
* 如果需要设置启动时JVM的参数，可以添加：environment: - JVM_OPTS=-Xmx12g -Xms12g -XX:MaxPermSize=1024m 用于配置jvm的参数。

ROOT.xml如下，tomcat启动后，部署的项目处于tomcat根目录
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Context path="" docBase="/home/xxxx/webroot/manage.war" reloadable="true">
</Context>
```
启动容器`docker-compose up -d`，通过在本地logs目录下可以查看容器启动的日志。
也可以通过`docker logs -f [CONTAINER ID]`查看容器的日志。

在启动容器后，如果需要进入容器，可以使用`docker exec -it [CONTAINER ID] bash`进入容器，这样可以用于排除项目启动以及生成的容器是否有问题。


参考：
* [docker官方文档](http://docs.docker-cn.com/get-started/)
* [docker之使用dockerfile配置tomcat、jdk环境](https://blog.csdn.net/qq_24557827/article/details/73729913)
* [指定jvm参数](https://segmentfault.com/a/1190000007271728)


