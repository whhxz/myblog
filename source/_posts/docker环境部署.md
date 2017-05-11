---
title: docker环境部署
date: 2017-05-11 18:31:56
tags: ['docker']
categories: ['docker']
---

## docker安装
使用系统Ubuntu 15.04 vivid，该版本无法安装最新版docker，需要升级ubuntu到最新版本。
安装最新版docker需要添加docker源，参考[Docker Community Edition for Ubuntu](https://store.docker.com/editions/community/docker-ce-server-ubuntu?tab=description)。
## 初步使用docker时出现问题
在使用docker build时强行停止后，在使用`docker image`出现如下错误
```
<none>              <none>              1c4475fbe64f        5 minutes ago       194 MB
```
使用`docker rmi 1c4475fbe64f`命令删除该无用image出现如下错误
```
Error response from daemon: conflict: unable to delete 1c4475fbe64f (must be forced) - image is being used by stopped container a4f0c195cff5
```
但是使用`docker ps`未显示任何数据
解决办法先`docker rm a4f0c195cff5` 之后`docker rmi 1c4475fbe64f`
