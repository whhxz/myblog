---
title: Golang编译安卓可执行文件
date: 2023-10-11 09:30:46
categories: ['Go']
tags: ['Go', 'Android', 'DDns', 'Termux']
---

一部旧手机在家发挥余热，安装Termux后，希望手机能自动更新DNS解析，方便远程访问家里网络。
最开始使用Python完成，为了方便能直接运行，用Go做了重新实现。

实现逻辑是，通过`https://4.ipw.cn`获取当前IP，然后通过阿里云SDK去检查IP是否一致，不一致则更新。

<!-- more -->
编译脚本
```bat
SET CGO_ENABLED=0
SET GOOS=android
SET GOARCH=arm64
go build -o .\out\ .\cmd\homeddns\
```
编译成功后在Termux上执行异常。
```log
Get "https://4.ipw.cn": dial tcp: lookup 4.ipw.cn on [::1]:53: read udp [::1]:48523->[::1]:53: read: connection refused
```
DNS解析的问题，问题原因可以看
> * https://github.com/termux/termux-app/issues/869#issuecomment-433985523
> * https://github.com/golang/go/issues/8877

解决办法，使用proot解决
```shell
pkg install proot
```
执行命令
```shell
proot -b $PREFIX/etc/resolv.conf:/etc/resolv.conf ./homeddns
```

> 希望golang log打印输出到文件，直接使用 >> ddns.log是不生效的，使用  2>> ddns.log生效
> * https://stackoverflow.com/questions/19965795/how-to-write-log-to-file

因为代码有阿里云授权key，所以不能写死在代码里。
解决办法是
在main方法里面定义变量
打包时添加参数
```bat
-ldflags "-X 'main.keyId=xxx' -X 'main.keySecret=xxx' -X 'main.recordId=xxx'"
```
