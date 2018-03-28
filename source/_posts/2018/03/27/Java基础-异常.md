---
title: Java基础-异常
date: 2018-03-27 20:34:58
categories: ['Java基础']
tags: ['基础', '异常']
---

在程序中经常会出现各种错误，在Java中可以通过异常进行展现出来。
在处理异常时，常用的关键字有：try、catch、finally、throw、throws；
try：通过会出现异常的代码块可以放入try代码块中，用于监听代码块中的是否会出现异常。
catch：通常用于捕获try代码块中出现的异常，用于发生异常捕获后做相关操作。
finally：用于try代码块执行完毕后，必须执行的地方，比如出现IOException后用于关闭连接，就算出现异常，finally也会执行。
throw：用于程序手动抛出异常。
throws：用于方法后，标识方法可能会出现异常。

在Java中异常分为两大类：Error、Exception。
Error：用来表示编译时和系统错误，出现这种错误时一般比较严重，一般有JVM抛出。
Exception：由程序本身可以处理的异常。

异常又可以分为：可检测异常、非检测异常
可检测异常：正确的程序在运行中，很容易出现的、情理可容的异常状况。编译器在编译代码时会检测该异常。一旦出现该异常，必须进行处理，也就是要么try...catch，要么throw抛出异常到上一层，由上层处理，如果一直上抛，最终会抛出到JVM层，最终导致JVM停止。除RuntimeException及其子类以外都是该异常，典型的如IOException。
非检测异常：这种异常并不会直接被编译器所检测，RuntimeException及其子类和Error。
<!-- more -->
除去Error，只看Exception分为运行时异常和非运行时异常：
运行时异常：和非检测异常一样，只是排除Error，都是RuntimeException及其子类。
非运行时异常：除RuntimeException及其子类以外的异常。

常见的异常关系继承图如下：
![](http://otxnth5wx.bkt.clouddn.com/20180328异常继承关系图.jpg)

更多异常知识：
[Java 异常处理的误区和经验总结](https://www.ibm.com/developerworks/cn/java/j-lo-exception-misdirection/)
[Java 异常处理及其应用](https://www.ibm.com/developerworks/cn/java/j-lo-exception/index.html)
