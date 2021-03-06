---
title: Java基础-枚举
date: 2018-03-23 10:04:07
categories: ['Java基础']
tags: ['基础', '枚举']
---

JDK1.5新增了枚举类型，定义一个枚举如下：
```java
public enum Main {
    MONDAY(1, "星期一"),
    TUESDAY(2, "星期二"),
    WEDNESDAY(3, "星期三"),
    THURSDAY(4, "星期四"),
    FRIDAY(5, "星期五"),
    SATURDAY(6, "星期六"),
    SUNDAY(7, "星期日");


    public int num;

    public String name;

    Main(int num, String name) {
        this.num = num;
        this.name = name;
    }

    public int getNum() {
        return num;
    }

    public String getName() {
        return name;
    }
}
```
<!-- more -->
通过命令`javap -p`查看编译后的class如下：
```javap
Compiled from "Main.java"
public final class Main extends java.lang.Enum<Main> {
  public static final Main MONDAY;
  public static final Main TUESDAY;
  public static final Main WEDNESDAY;
  public static final Main THURSDAY;
  public static final Main FRIDAY;
  public static final Main SATURDAY;
  public static final Main SUNDAY;
  public int num;
  public java.lang.String name;
  private static final Main[] $VALUES;
  public static Main[] values();
  public static Main valueOf(java.lang.String);
  private Main(int, java.lang.String);
  public int getNum();
  public java.lang.String getName();
  static {};
}
```
从反编译的代码可以看出：
1、最终生成的是final修饰类，`java.lang.Enum`的子类
2、之前定义的枚举变为了当前类的对象
3、构造方法是私有的
4、新增values、valueOf方法

通过类通过final修饰禁止继承，提前定义好相应的对象，private修饰构造方法禁止创建新的对象。

在创建枚举类时，会调用父类Enum构造方法，传入枚举的名称、枚举的序数（用于查看枚举声明的数组中的位置）。
Enum因为实现了Comparable类，所以枚举可以进行比较，看源码得知先比较的是某个类，之后比较是枚举的序数是否相等。

枚举抽象方法用法：
```java
public enum Main {
    SATURDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    },
    SUNDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    };

    public abstract void doSomeThing();
}
```
通过定义一个抽象方法，然后在定义枚举是需要实现改方法（反编译后生成的类为抽象类，枚举值为其内部类），既然是类，那么如果定义的是普通方法，那么可以重写定义的方法。

枚举因为编译器最终生成的是类，在Java中不支持多继承（已经继承java.lang.Enum），但是可以实现接口，如下：
```java
interface IDeme {
    void doSomething();
}

public enum Main implements IDeme {
    SATURDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    },
    SUNDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    }
}
```

枚举除了在定义唯一的属性，同时在使用单例设计模式时，保证了唯一性。
普通的单例在通过序列化、反序列化时，可以生成一个新的对象（通过添加readResolve方法来解决该问题）；也可以通过反射来强行调用私有构造方法生成新的对象。

在对枚举序列化和反序列化过程中，仅仅是把枚举对象的name序列化，反序列化的时候是通过Enum.valueOf()来通过name找到对应的对象。同时编译器不允许对这种序列化机制做定制，所有禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法。
参考：[1.12 Serialization of Enum Constants](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html)

举例如下：
```java
interface IDeme {
    void doSomething();
}

public enum Main implements IDeme {
    SATURDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    },
    SUNDAY() {
        @Override
        public void doSomething() {
            System.out.println("nothing");
        }
    };

    public static void main(String[] args)throws Exception {
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(System.out);
        Main saturday = Main.SATURDAY;
        objectOutputStream.writeObject(saturday);
        //输出类似 乱码com.whh.netty.Main乱码java.lang.Enum乱码xptSATURDAY
        objectOutputStream.close();
    }
}
```
从上面可以依稀看出最后是SATURDAY字符串，可以通过验证反序列化比较是否一致。
Enum.valueOf如下：
```java
public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                            String name) {
    T result = enumType.enumConstantDirectory().get(name);
    if (result != null)
        return result;
    if (name == null)
        throw new NullPointerException("Name is null");
    throw new IllegalArgumentException(
        "No enum constant " + enumType.getCanonicalName() + "." + name);
}
```
通过传入的类调用enumConstantDirectory，该方法返回枚举名字的常量（其实就是反射获取枚举类中values返回枚举中定义的数组），通过key或获取到对应的枚举值。这样就保证枚举返回的事唯一值。

通过反射创建新枚举：
```java
public enum Main {
    SATURDAY,
    SUNDAY;

    public static void main(String[] args) throws Exception {
        Constructor<Main> declaredConstructor = Main.class.getDeclaredConstructor(String.class, int.class);
        declaredConstructor.setAccessible(true);
        Main test = declaredConstructor.newInstance("test", 11);//Cannot reflectively create enum objects
        System.out.println(test);
    }
}
```
打开Constructor.newInstance源码看到
```java
if ((clazz.getModifiers() & Modifier.ENUM) != 0)
    throw new IllegalArgumentException("Cannot reflectively create enum objects");
```
如果是枚举类型会直接抛出异常。

在使用枚举过程中，还有EnumSet、EnumMap辅助类帮助快速使用，EnumSet使用了位向量，后续位运算时分析。

参考：[深入理解Java枚举类型(enum)](https://blog.csdn.net/javazejian/article/details/71333103)
