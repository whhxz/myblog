---
title: Java基础-String
date: 2018-03-22 10:19:49
categories: ['Java基础']
tags: ['基础', 'String']
---

平常基本类型+String是在代码中使用最多的地方。分析下String特别的地方。

### 字符串替换
String的字符串替换有3个方法：replace、replaceFirst、replaceAll

#### replace
```java
public String replace(char oldChar, char newChar)

public String replace(CharSequence target, CharSequence replacement)
```
String.replace有两个重载方法，一个参数是char、一个是CharSequence
第一个replace是通过原有String中char数组生成一个新数组，然后把新数组中的oldChar替换为newChar，返回一个新的String

* 注：如果oldChar==newChar返回的是this。

第二个relace参数是CharSequence，String是其实现类，所以平常一般传的参数是String（不过StringBuffer、StringBuilder也是该接口实现类）。该replace实现虽然是通过Pattern，但是在使用Pattern的时候，设置了`Pattern.LITERAL和Matcher.quoteReplacement`，所以转义是不起效的。

所以对于replace而言，就是 **替换所有** 符合的字符，无转义，无正则表达式（设置Pattern转义失效）。
<!-- more -->
#### replaceAll
```java
public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}
```
replaceAll就是纯粹的通过正则表达式替换

#### replaceFirst
```java
public String replaceFirst(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}
```
再使用replace或者replaceAll的时候，是替换所有匹配的字符串，如果不需要匹配全部，可以通过replaceFirst尝试值替换第一个。不过replaceFirst的匹配模式也是通过正则表达式。

```java
//示例
String str = "com.whh.String";

System.out.println(str.replace('.', '/'));//  com/whh/String
System.out.println(str.replace(".", "$0"));// com$0whh$0String
System.out.println(str.replaceAll(".", "$0"));//  com.whh.String
System.out.println(str.replaceFirst(".", "$0"));//  com.whh.String
System.out.println(str.replace(".", "/"));//  com/whh/String
System.out.println(str.replaceAll(".", "/"));// //////////////
System.out.println(str.replaceFirst(".", "/"));// /om.whh.String
```

参考：[java字符串的替换replace、replaceAll、replaceFirst的区别详解](https://my.oschina.net/u/816576/blog/369643)

### String "+"
在Java中是不支持运算符重载的，但是在平常使用过程中拼接字符串时，我们都是通过`+`来对字符串做相关操作，这是一种错觉。
```java
//例子
public class Main{
    public static void main(String[] args){
        String a = "com";
        String b = ".";
        String name = a + b + "whh" + b + "test";
    }

}
```
在上述代码中使用的`+`拼接的字符串，之后通过`javac`编译代码生成class文件，通过命令`javap -c Main.class`对class进行反汇编。得到如下：
```java
Compiled from "Main.java"
public class Main {
  public Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String com
       2: astore_1
       3: ldc           #3                  // String .
       5: astore_2
       6: new           #4                  // class java/lang/StringBuilder
       9: dup
      10: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
      13: aload_1
      14: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      17: aload_2
      18: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      21: ldc           #7                  // String whh
      23: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      26: aload_2
      27: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      30: ldc           #8                  // String test
      32: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      35: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      38: astore_3
      39: return
}
```
上述返回编译后可以看到编译成Class后实际是通过StringBuilder来进行拼接的字符串。
如果代码修改为如下：
```java
public class Main{
    public static void main(String[] args){
        String a = "com";
    		for (int i = 0; i <=10; i++){
    			a += a;
    		}
    }
}
```
在for循环中拼接字符串，通过反编译可以得到：
```java
Compiled from "Main.java"
public class Main {
  public Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String com
       2: astore_1
       3: iconst_0
       4: istore_2
       5: iload_2
       6: bipush        10
       8: if_icmpgt     36
      11: new           #3                  // class java/lang/StringBuilder
      14: dup
      15: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      18: aload_1
      19: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      22: aload_1
      23: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      26: invokevirtual #6                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      29: astore_1
      30: iinc          2, 1
      33: goto          5
      36: return
}
```
可以看Code标记33，goto5，这里表示for循环，在Code标记11地方会创建一个StringBuilder，也就是每次for循环都会创建一个StringBuilder对象，比较浪费，可以通过在for循环外卖创建StringBuilder后在for循环内通过append拼接字符串。

参考：[Java 深究字符串String类(1)之运算符"+"重载](http://blog.csdn.net/dextrad_ihacker/article/details/53055709)

### String不可变
在String中通过char[]表达一个字符串，char数组被final修饰，这也表示了单String被初始化后char数组就不能在被修改。而且String类也是final修饰，表示该类不可被继承。
正常情况下不可以修改，不过因为final修饰的是数组，数组引用不可修改，但是数组的值却可以修改，可以通过反射修改char数组中的值。
```java
public static void main(String[] args) throws Exception {
    Set<String> set = new HashSet<>();
    String a = "whh";
    String b = "www";
    set.add(a);
    set.add(b);
    Field value = String.class.getDeclaredField("value");
    value.setAccessible(true);
    char[] chars = (char[]) value.get(b);
    Array.set(chars, 1, 'h');
    Array.set(chars, 2, 'h');
    value.setAccessible(false);
    for (String s : set) {
        System.out.printf("%s hash is %d\n", s, s.hashCode());
    }
    //whh hash is 117687
    //whh hash is 118167
}
```
通过反射修改String内char数组的值，最终输出的字符串一样，但是其hashCode不一样。

把String设计为不可变有几大优点：
1、在Java内存中存在一块常量池，`String a = "whh";new String("whh").intern()`这种创建的对象会存入常量池中，当再次创建一个相同的对象时，会直接指向常量池中的地址，因为String在使用过程中非常频繁，可以通过常量池复用相同的字符串。
2、在String创建完毕后，该字符串的hash在创建时就已经计算完毕，方便直接使用，特别是Set中特别有用。如果String可变，在使用Set时，如果先存入`"aaa", "bbb"`，之后修改`"aaa"`对象为`"bbb"`，因为String可变，所以在Set中现在是`"bbb", "bbb"`，出现了重复数据。
3、并发安全，因为String不可变，所有可以在多线程中共享。

注：String.intern是如果常量池存在着返回常量池中的对象，如果常量池不存在则把当前对象放入常量池（在常量池判断存在时类似equals比较）。

在平常使用`String a="whh"`时，创建的String存在与常量池中，如果使用`new String("whh")`，因为使用了`new`关键字，所以会告诉JVM在堆中开辟一块内存，用于存放创建的对象，所以在使用`==`时比较的事内存地址，一个在堆中一个在常量池中所以不会相等。
