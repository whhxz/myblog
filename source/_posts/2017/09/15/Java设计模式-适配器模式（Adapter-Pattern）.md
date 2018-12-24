---
title: Java设计模式-适配器模式（Adapter Pattern）
date: 2017-09-15 14:12:06
categories: ['设计模式']
tags: ['设计模式', '适配器模式', 'Adapter Pattern']
---
### 不兼容结构的协调--适配器模式
> 适配器模式把一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作。

当客户类调用适配器的方法时，在适配器类的内部将调用适配者类的方法，而这个过程对客户类是透明的，客户类并不直接访问适配者类。因此，适配器让那些由于接口不兼容而不能交互的类可以一起工作。适配器的特点在于兼容。
![](http://image.whhxz.smallstool.cn/20170918%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-18%E4%B8%8B%E5%8D%883.17.05.png)
* Target是用户使用的目标类
* Adaptee是需要适配的第三方类
* Adapter是适配器类
<!-- more -->

举例如下：
![](http://image.whhxz.smallstool.cn/20170918%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-18%E4%B8%8B%E5%8D%883.11.35.png)
```java
/**
 * 插头
 */
interface Plug{
    /**
     * 插入充电
     */
    void charge();
}

/**
 * 双孔插头
 */
class DoubleEndedPlug implements Plug{

    @Override
    public void charge() {
        System.out.println("双孔插头充电");
    }
}

/**
 * 3孔插座
 */
class ThreeHoleSocket{
    public void charge(){
        System.out.println("三孔插座充电");
    }
}
/**
 * 对象适配器
 */
class DoublePlugAdapter extends DoubleEndedPlug{
    private ThreeHoleSocket socket;

    public DoublePlugAdapter(ThreeHoleSocket socket) {
        this.socket = socket;
    }

    @Override
    public void charge() {
        //封装数据等操作，使适配器适配
        System.out.println("双孔适配插入三孔插座");
        socket.charge();
        //封装返回值等操作，使适配器适配
    }
}


public class Main {
    public static void main(String[] args) {
        Plug plug = new DoubleEndedPlug();
        //默认双孔充电
        plug.charge();
        Plug plugAdapter = new DoublePlugAdapter(new ThreeHoleSocket());
        //使用适配器三孔充电
        plugAdapter.charge();
    }
}
```
使用适配器后，可以使用双孔充电器插入双孔插座充电。
在JDK中InputStreamReader(InputStream)就是使用的装饰适配器设计模式
```java
//字节流
InputStream fileInputStream = new FileInputStream(new File("tmp.txt"));
StringBuilder sb = new StringBuilder();
int tmp;
while ((tmp = fileInputStream.read()) != -1){
    sb.append((char)tmp);
}
System.out.println(sb);
fileInputStream.close();

//使用适配器模式，转换为字符流，读取单个字符
InputStream inputStream = new FileInputStream(new File("tmp.txt"));
InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
sb = new StringBuilder();
while ((tmp = inputStreamReader.read()) != -1){
    sb.append((char)tmp);
}
System.out.println(sb);
```
InputStreamReader父类Reader是面向字符流的读取接口，使用适配器设计模式后，从字节流中读取数据。

### 类适配器
除了对象适配器模式之外，还有类适配器。类适配器模式和对象适配器模式最大的区别在于适配器和适配者之间的关系不同，对象适配器模式中适配器和适配者之间是关联关系，而类适配器模式中适配器和适配者是继承关系。
```java
/**
 * 插头
 */
interface Plug{
    /**
     * 插入充电
     */
    void charge();
}

/**
 * 双孔插头
 */
class DoubleEndedPlug implements Plug{

    @Override
    public void charge() {
        System.out.println("双孔插头充电");
    }
}

/**
 * 3孔插座
 */
class ThreeHoleSocket{
    public void charge(){
        System.out.println("三孔插座充电");
    }
}

/**
 * 类适配器
 */
class DoublePlugAdapter extends ThreeHoleSocket implements Plug{

    @Override
    public void charge() {
        //封装数据等操作，使适配器适配
        System.out.println("双孔适配插入三孔插座");
        super.charge();
        //封装返回值等操作，使适配器适配
    }
}


public class Main {
    public static void main(String[] args) {
        Plug plug = new DoubleEndedPlug();
        //默认双孔充电
        plug.charge();
        Plug plugAdapter = new DoublePlugAdapter();
        //使用适配器三孔充电
        plugAdapter.charge();
    }
}
```
因为java不支持多继承，在使用类适配器的时候，会受到很多限制。如果目标类没有接口，无法使用类适配器。

### 双向适配器
双向适配器其实就是在适配器中同时拥有双方的对象，只是对适配器的一种扩充，使用相对较少。

### 缺省适配器
缺省适配器是适配器模式的一种变种。
> 当不需要实现一个接口所提供的所有方法时，可先设计一个抽象类实现该接口，并为接口中每个方法提供一个默认实现（空方法），那么该抽象类的子类可以选择性地覆盖父类的某些方法来实现需求，它适用于不想使用一个接口中的所有方法的情况，又称为单接口适配器模式。

![](http://image.whhxz.smallstool.cn/20170918%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-18%E4%B8%8B%E5%8D%883.27.08.png)

如java.awt.event中WindowAdapter就是使用的缺省适配器设计模式
![](http://image.whhxz.smallstool.cn/20170918%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-18%E4%B8%8B%E5%8D%883.29.11.png)
![](http://image.whhxz.smallstool.cn/20170918%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-18%E4%B8%8B%E5%8D%883.29.18.png)

在处理窗口事件的时候，可以使用WindowListener接口，但是需要实现接口中所有方法，但是在使用过程中不需要，所以就可以使用WindowAdapter来完成相同的事情。WindowAdapter中实现的方法都为空方法。

### 适配器设计模式总结
优点：
* 适配器设计模式是讲现有方法转换为所需要的方法，实现了对类的复用。
* 适配器实现了目标类和适配者的解耦
* 修改适配器可以在不修改源代码的情况下实现，符合“开闭原则”
缺点：
* 过多使用适配器会导致系统混乱。
* final无法适配
* 不支持多继承，适配有限制
适配器一般多用于后期系统扩展、修改。
