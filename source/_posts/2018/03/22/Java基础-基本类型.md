---
title: Java基础-基本类型
date: 2018-03-22 17:40:43
categories: ['Java基础']
tags: ['基础', '基本类型']
---

JDK1.5中引入了自动拆装箱，方便基本类型和对象的转换。

| 基本类型 | 对象 |
| ---- |:----- |
| byte | Byte|
|short | Short|
|char | Character|
|int | Integer|
|long | Long|
|float| Float|
|double| Double|
|boolean| Boolean|

自动拆箱：基本类型转换为对象，一般通过类的静态方法`*Value()`转换，如Integer中`Integer.intValue(int)`;
自动装箱：把对象转换为相应的基本类型，一般是`对象.valueOf()`。
<!-- more -->
自动拆装箱一般发生在基本类型和对象相互转换的时候。

```Java
public class Main1{
	public void main(String[] arg){
		int sum = 1;
		Integer i = sum;
		int b = i;
	}
}
```
使用`javap -c`反编译后：
```javap
Compiled from "Main1.java"
public class Main1 {
  public Main1();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: istore_2
       2: iload_2
       3: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       6: astore_3
       7: aload_3
       8: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
      11: istore        4
      13: return
}
```
可以看到编译后使用的是`Integer.valueOf`和`Integer.intValue`。

自动拆装箱的时候有几个需要注意的地方：
#### 方法重载
```java
public class Main {
    public static void main(String[] args) throws Exception {
        sout(1);
        sout(new Integer(1));
    }

    public static void sout(int i){
        System.out.printf("int %d\n", i);
    }
    public static void sout(Integer i){
        System.out.printf("Integer %d\n", i);
    }
}
```
在使用int和Integer重载时，并不会自动拆装箱

#### 对象多余的创建
```java
Integer sum = 0;
for(int i = 0; i < 10000; i++){
  sum+=1;
}
```
这种在反编译后可以每次相加的时候都会出现拆装箱，因为在Java中没有运算符重载，所以需要不同的解析拆装箱。

#### 对象缓存
Integer自动装箱时，使用的事Integer.valueOf()，源码如下：
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
int在`-128~127`访问时，使用的Integer内缓存的值，这样就会出现如下问题：
```java
Integer a = 100;
Integer b = 100;

Integer c= 1000;
Integer d = 1000;
System.out.println(a == b);// true
System.out.println(c == d);// false
```
注：在Double、Float并不会缓存。

#### == 和 equels
因为对象不支持四则运算操作，所以在对象和基本类型运算时会发生拆箱；在对象和基本类型`==`比较时会发生拆箱；使用`equels`比较时，会发生装箱，因为equels参数为Object，虽然发生了装箱但是因为Integer重写了equels，比较Integer里面的值。


参考：[Java中的自动装箱与拆箱](https://droidyue.com/blog/2015/04/07/autoboxing-and-autounboxing-in-java/)
