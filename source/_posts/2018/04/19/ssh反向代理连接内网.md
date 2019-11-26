---
title: ssh反向代理连接内网
date: 2018-04-19 11:29:55
categories: ['ssh']
tags: ['ssh', '反向代理', '内网穿透']
---

因为在家有个笔记本装的ubuntu，需要在公司访问家里笔记本，家中的宽带没有内网IP，所以需要通过反向代理访问家中电脑，需要一台外网服务器作为中转，使用的是阿里云做为中转服务器。

| 机器 | 描述 |
| : --| : --|
| 家ubuntu | 内网 |
| 阿里云 | 独立IP |
| 公司 | 内网 |

1、先通过家里机器反向代理到阿里云
2、阿里云提供正向代理，通过公司访问

<!-- more -->

反向代理`ssh -fCNR`
正向代理`ssh -fCNL`
> -f 后台执行ssh指令
-C 允许压缩数据
-N 不执行远程指令
-R 将远程主机(服务器)的某个端口转发到本地端指定机器的指定端口
-L 将本地机(客户机)的某个端口转发到远端指定机器的指定端口
-p 指定远程主机的端口

在家中电脑执行：
`ssh -fnNT -R 6666:localhost:22 aliyun@xxx.xx.xx.xx`
> 上述中6666表示阿里云监听的本地端口

登陆阿里云，`ssh -p6666 homeuser@localhost`即可登陆家中服务器。

现在需要进一步使用正向代理，方便公司通过代理访问。

在阿里云上执行
`ssh -fCNL *:7777:localhost:6666 localhost`
> 上述6666表示之前分析代理监听的端口，7777表示正向代理的端口

在公司使用`ssh -p7777 homeuser@xxx.xx.xx.xx`即可登陆家中电脑。

因为ssh可能会出现超时端口，所以在家中启动的反向代理使用autossh，如下：
```sh
autossh -M 6667 \
-fN -o "PubkeyAuthentication=yes" \
-o "StrictHostKeyChecking=false" -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" \
-R 6666:localhost:22 aliyun@xxx.xx.xx.xx
```
> 6667 用于重连

端口监听命令：
netstat -tunlp

参考：
* [利用ssh反向代理以及autossh实现从外网连接内网服务器](https://www.cnblogs.com/kwongtai/p/6903420.html)
* [SSH 远程端口转发](https://lvii.github.io/system/2013/10/08/ssh-remote-port-forwarding/)