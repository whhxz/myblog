---
title: 一次老旧vue2项目docker启动
date: 2023-12-15 15:44:58
categories:
tags: ['vue2', 'docker']
---

为了兼容老旧电脑，公司前端统一使用的vue2，项目种采用scss的老旧版本，导致当前win10版本需要python2来编译，本地电脑有一堆python3和node18项目，不想破环当前项目环境，所有打算通过docker启动。

docker容器镜像采用的是**node:14.14-alpine3.10**，在[Alpine's Python 2 Package is Deprecated ](https://github.com/nvm-sh/nvm/issues/2895)有说明alpine 3.12已经不使用python2了，所以直接使用3.10版本。

<!-- more -->
```xml
version: '3.8'
services:
  mongo:
    container_name: node14
    image: node:14.14-alpine3.10
    restart: always
    environment:
      - TZ=Asia/Shanghai
    ports:
      - '127.0.0.1:13000:3000'
    volumes:
      - ./data/:/opt/web/
    network_mode: bridge
    tty: true
```
使用**tty**是伪终端，启动后通过shell连接到项目做相关操作。

进入容器后，需要通过`apk add`安装：
* python2
* make
* g++
  
之后再通过yarn安装依赖。通过`yarn serve`启动项目