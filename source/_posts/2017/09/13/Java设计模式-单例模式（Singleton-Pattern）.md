---
title: Java设计模式-单例模式（Singleton Pattern）
date: 2017-09-13 14:31:47
categories: ['设计模式']
tags: ['设计模式', '单例模式', 'Singleton Pattern']
---

### 单例模式概念
> * 有且只有一个实例
> * 必须自行创建这个实例
> * 必须自行向整个系统提供这个实例

### 单例设计
单例的创建一般分为 **饿汉式单例** 、 **懒汉式单例** 。
#### 饿汉式单例
```java
class EagerSingleton {
    //自己创建的实例
    private static EagerSingleton singleton = new EagerSingleton();

    /**
     * 私有构造函数，不允许外部创建
     */
    private EagerSingleton() {
    }

    /**
     * 对外提供当前实例
     * @return
     */
    public static EagerSingleton getInstance() {
        return singleton;
    }
}
```
<!-- more -->
当类被加载的时候，static会初始化，创建当前对象，因为构造函数是私有的，这样避免了外部创建改对象，通过静态方法提供了系统外部的访问。由java类加载器保证该类只会加载一次。采用的是空间换时间。
缺点：该类就是在系统中不使用也会初始化，对象会一直存在，不会被回收。占用系统资源。

在jdk中有啥用饿汉式单例设计模式，如Runtime：
![](/images/old/20170913%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-13%E4%B8%8B%E5%8D%883.02.15.png)

#### 懒汉式单例
```java
/**
 * 懒汉式单例
 */
 /**
  * 懒汉式单例
  */
 class LazySingleton {
   /**
    * 避免jdk做相关优化，导致代码重排序
    */
   private volatile static LazySingleton singleton = null;

     /**
      * 私有构造函数
      */
     private LazySingleton() {
     }

     /**
      * 创建获取单例对象
      * @return
      */
     public static LazySingleton getInstance() {
         //懒汉式改进，避免方法上锁导致效率变低
         if (singleton == null) {
             //双重检查锁，保证创建的时候只有一个线程参与
             synchronized(LazySingleton.class){
                 if (singleton == null){
                     singleton = new LazySingleton();
                 }
             }
         }
         return singleton;
     }
 }
```
类被初始化的时候，不会创建对象。在第一次调用getInstance才会创建对象。第一次判断，判断是否单例已经存在。在创建改单例的时候，加锁后判断，是避免多线程访问的时候，保证只创建一个实例。
在java中java.util.Calendar使用的是懒汉式。
缺点：因为使用了volatile和判断，速度相对饿汉式要慢，懒汉式避免了未使用的时候占用系统资源。

#### Initialization Demand Holder (IoDH) 单例
饿汉式单例和懒汉式单例创建各有优缺点。使用IoDH可以避免二者缺点
```java
class Singleton{
    /**
     * 私有构造函数
     */
    private Singleton(){}

    /**
     * 静态内部类，用于创建单例
     */
    private static class HolderClass{
        private final static Singleton singleton = new Singleton();
    }

    /**
     * 对外提供单例
     * @return
     */
    public static Singleton getInstance(){
        return HolderClass.singleton;
    }
}
```
在类被初始化的时候，静态内部类并不会被初始化，只有使用到静态内部类的时候，才会被初始化。因为是私有类，保证了外部无法访问，静态属性由java类加载器保证了该类只被初始化一次。同时没有锁没有判断，集合之前两种单例创建的优点。

#### 枚举单例
之前三种单例创建有个缺点，在单例实现序列化后，在实例序列化反序列化后，会存在多个实现，违背了单例设计模式。例：
```java
class Singleton implements Serializable{
    private static final long serialVersionUID = -1218018069415776722L;

    /**
     * 私有构造函数
     */
    private Singleton(){}

    /**
     * 静态内部类，用于创建单例
     */
    private static class HolderClass{
        private final static Singleton singleton = new Singleton();
    }

    /**
     * 对外提供单例
     * @return
     */
    public static Singleton getInstance(){
        return HolderClass.singleton;
    }
}

public class Main {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        //创建单例
        Singleton instance = Singleton.getInstance();
        //创建输入输出管道
        PipedOutputStream outputStream = new PipedOutputStream();
        PipedInputStream inputStream = new PipedInputStream();
        inputStream.connect(outputStream);
        //准备对象序列化
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
        objectOutputStream.writeObject(instance);

        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        Singleton singleton = (Singleton) objectInputStream.readObject();
        outputStream.close();
        inputStream.close();

        System.out.println(singleton == instance);//false
    }
}
```
注：可以使用readResolve避免，但是需要自己处理

在该例子中，序列化后的对象是一个新的实例。
解决办法使用枚举做单例：
```Java
enum EnumSingleton{
    SINGLETON;
    public void doSomeThing(){
        System.out.println("单例");
    }
}
```
在《高效Java 第二版》中说：单元素的枚举类型已经成为实现Singleton的最佳方法。用枚举来实现单例非常简单，只需要编写一个包含单个元素的枚举类型即可。
枚举在序列化时候其实是序列化了name，在反序列化的时候，通过valueOf(name)，保证了枚举的唯一性。

### 单例模式总结
单例因为控制实例的创建，在系统内存中只有唯一一份，可以节约系统资源，减少了对象的频繁创建和销毁。在Spring中默认创建的对象为单例，在Struts2中action默认创建的对象为多例。
单例缺点在于扩展比较困难，而且违背了“单一职责原则”，因为单例又是对象创建工厂，又是实例，还包含相关业务方法。
单例设计模式用在，系统因为消耗太大只允许创建一个对象情况。或者系统只能使用一个公共访问点，除了公共访问点，不能通过其他途径访问该实例。
