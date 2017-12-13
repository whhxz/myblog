---
title: Docker 安装
date: 2017-10-10 18:24:21
categories: ['Docker']
tags: ['Docker']
---

### 安装虚拟机
本机使用vmware fusion安装CentOS7，配置好网络后打开虚拟机。

因为docker无法被墙，需要使用代理。
当前用户代理修改~/.bash_profile添加
```sh
#代理
http_proxy=http://proxy.com:8080
https_proxy=http://proxy.com:8080
export http_proxy
export https_proxy
```
执行命令：<!-- more -->
```sh
source ~/.bash_profile
```
如果需要所有用户代理生效需要修改/etc/profile

### 安装Docker
[CentOS Docker官方安装文档](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)

每次操作docker都需要sudo过于麻烦，把当前用户添加到docker组
> 1.创建docker组（可能提示已经存在docker组）：sudo groupadd docker
> 2.当前用户加入docker组：sudo gpasswd -a ${USER} docker
> 3.重启docker服务：sudo systemctl restart docker
> 4.刷新docker成员：newgrp - docker

之后操作docker就不要添加sudo了。

### 配置Docker国内加速
使用阿里云加速Docker
打开[Docker Hub镜像站点](https://cr.console.aliyun.com/#/accelerator)获取个人加速地址。
通过修改daemon配置文件/etc/docker/daemon.json来使用加速器：
```sh
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://w5g1eln1.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
