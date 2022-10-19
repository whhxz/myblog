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
![](/images/old/20180328异常继承关系图.jpg)


# 异常流程
编写代码如下：
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(ex());
    }

    public static int ex(){
        try {
            int i = 1/0;
        }catch (Exception e){
           // return 1;
			throw new RuntimeException("aaa");
        }finally {
            return 0;
        }
    }
}
```
`javap -v -l -p -c Main.class`反编译后只看部分代码
```javap
public static int ex();
descriptor: ()I
flags: ACC_PUBLIC, ACC_STATIC
Code:
    stack=3, locals=2, args_size=0
    0: iconst_1
    1: iconst_0
    2: idiv
    3: istore_0
    4: iconst_0
    5: ireturn
    6: astore_0
    7: new           #6                  // class java/lang/RuntimeException
    10: dup
    11: ldc           #7                  // String aaa
    13: invokespecial #8                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
    16: athrow
    17: astore_1
    18: iconst_0
    19: ireturn
    Exception table:
        from    to  target type
            0     4     6   Class java/lang/Exception
            0     4    17   any
            6    18    17   any
    LineNumberTable:
    line 8: 0
    line 13: 4
    line 9: 6
    line 11: 7
    line 13: 17
    StackMapTable: number_of_entries = 2
    frame_type = 70 /* same_locals_1_stack_item */
        stack = [ class java/lang/Exception ]
    frame_type = 74 /* same_locals_1_stack_item */
        stack = [ class java/lang/Throwable ]
```
上述代码中可以看Exception table中 `0~4`行如果抛出了`java/lang/Exception`异常，那么转跳到第6行，`0~4`如果发生了其他异常转跳到17行，如果`6~18`行发生其他异常转跳17行。

上述代码中try包含的代码块就是反编译后`0~4`行，转到第6行为catch异常代码，其中我们是throw一个异常，其实就是创建一个异常对象，然后执行`athrow`命令，之后去异常表里面查找是否存在异常处理，之后转跳到17行，继续执行然后返回。

这里就可以看出上述代码，并不会抛出`RuntimeException`，直接返回finlly中的值。

如果修改代码如下：
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(ex());
    }

    public static int ex(){
        try {
            int i = 1/0;
        }catch (Exception e){
            throw new RuntimeException("aaa");
        }finally {
            System.out.println("finally");
        }
        return 1;
    }
}
```
反编译后得到：
```javap
public static int ex();
descriptor: ()I
flags: ACC_PUBLIC, ACC_STATIC
Code:
    stack=3, locals=2, args_size=0
    0: iconst_1
    1: iconst_0
    2: idiv
    3: istore_0
    4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
    7: ldc           #5                  // String finally
    9: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    12: goto          37
    15: astore_0
    16: new           #8                  // class java/lang/RuntimeException
    19: dup
    20: ldc           #9                  // String aaa
    22: invokespecial #10                 // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
    25: athrow
    26: astore_1
    27: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
    30: ldc           #5                  // String finally
    32: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    35: aload_1
    36: athrow
    37: iconst_1
    38: ireturn
    Exception table:
        from    to  target type
        0     4    15   Class java/lang/Exception
        0     4    26   any
        15    27    26   any
```
反编译后`0~4`中如果出现异常转跳到15，如果`15~27`中出现异常转跳26，在`15~27`为catch中语句，创建RuntimeException对象，然后`athrow`抛出异常，转跳到26，执行finally语句。之后代码继续执行，可以看到在36处有个`athrow`，但是在异常表中并没有该处异常处理，所以异常会被抛出。

在java中通过`方法调用栈`来跟踪线程调用一系列方法调用过程，改栈堆保存了每个调用方法的本地信息（比如方法的局部变量）。每个线程都有一个独立的方法调用栈。每当一个新方法被调用时，Java虚拟机把描述该方法的栈结构置入栈顶，位于栈顶的方法为正在执行的方法。方法执行完毕后该方法会被弹出栈顶。如果在方法中出现异常，且异常表中没有该异常处理，在异常抛出后，虚拟机会从方法栈中弹出该方法，然后在前一个调用方法处查询是否有该异常处理，如果一直没有就会一直追溯到调用栈底部，这时会如下处理：
1、调用异常对象的printStackTrace()方法，打印来自方法调用栈的异常信息。
2、如果该线程不是主线程，那么终止这个线程，其他线程继续正常运行。如果该线程是主线程（即方法调用栈的底部为main()方法），那么整个应用程序被终止。

* `Thread.currentThread().getStackTrace()`可以打印当前方法调用栈。


更多异常知识：
[Java 异常处理的误区和经验总结](https://www.ibm.com/developerworks/cn/java/j-lo-exception-misdirection/)
[Java 异常处理及其应用](https://www.ibm.com/developerworks/cn/java/j-lo-exception/index.html)
[从字节码的角度来看try-catch-finally和return的执行顺序](https://blog.csdn.net/u010412719/article/details/50043865)
[java学习笔记《面向对象编程》——异常处理](https://blog.csdn.net/dnxyhwx/article/details/6975087)
