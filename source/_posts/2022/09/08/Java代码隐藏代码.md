---
title: Java代码隐藏代码
date: 2022-09-08 16:19:41
categories: ['Java']
tags: ['小技巧']
---

Java代码中可以通过Unicode注释后隐藏部分实际代码
```java
@Test
public void demo(){
    // \u000d System.out.println("Hello");
}
```
上述代码会输出**Hello**，因为前面unicode会转义为换行，后面的代码正常执行。后面代码也可以全部转义为unicode，用于隐藏，如下。
```java
@Test
public void demo(){
    // \u000d \u0053\u0079\u0073\u0074\u0065\u006d\u002e\u006f\u0075\u0074\u002e\u0070\u0072\u0069\u006e\u0074\u006c\u006e\u0028\u0022\u0048\u0065\u006c\u006c\u006f\u0022\u0029\u003b
    \u0053\u0079\u0073\u0074\u0065\u006d\u002e\u006f\u0075\u0074\u002e\u0070\u0072\u0069\u006e\u0074\u006c\u006e\u0028\u0022\u0048\u0065\u006c\u006c\u006f\u0022\u0029\u003b
}
```
上述会输出2次**Hello**。
