---
title: Java设计模式-代理模式（Proxy-Pattern）
date: 2017-10-18 09:21:16
categories: ['设计模式']
tags: ['设计模式', '代理模式', 'Proxy-Pattern']
---

### 代理模式
>给某一个对象提供一个代理或占位符，并由代理对象来控制对原对象的访问。


代理模式简单UML如下：
![](http://otxnth5wx.bkt.clouddn.com/20171018屏幕快照2017-10-18上午9.43.05.png)

* 接口：它声明了其实现和代理的共同接口，这样一来在任何使用其实现类的地方都可以使用代理类，客户端通常需要针对接口进行编程（cglib例外）。
* 代理（proxy）：它包含了对实现类的引用，从而可以在任何时候操作实现类对象；在代理类中提供一个与实现类角色相同的接口，以便在任何时候都可以替代接口实现类；代理类还可以控制对实现类的使用，负责在需要的时候创建和删除实现类对象，并对实现类对象的使用加以约束。通常，在代理类中，客户端在调用所引用的实现类操作之前或之后还需要执行其他操作，而不仅仅是单纯调用实现类对象中的操作。
* 实现类：接口的具体实现，定义了客户端真实方法。

代理模式是常用的结构型设计模式之一，当无法直接访问某个对象或访问某个对象存在困难时可以通过一个代理对象来间接访问，为了保证客户端使用的透明性，所访问的真实对象与代理对象需要实现相同的接口。根据代理模式的使用目的不同，代理模式又可以分为多种类型，例如保护代理、远程代理、虚拟代理、缓冲代理等，它们应用于不同的场合，满足用户的不同需求。代理的实现又分为静态代理、动态代理。

* 远程代理(Remote Proxy)：为一个位于不同的地址空间的对象提供一个本地的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在另一台主机中，远程代理又称为大使(Ambassador)。（如：SOA、Web Service）
* 虚拟代理(Virtual Proxy)：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。
* 保护代理(Protect Proxy)：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。
* 缓冲代理(Cache Proxy)：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果。
* 智能引用代理(Smart Reference Proxy)：当一个对象被引用时，提供一些额外的操作，例如将对象被调用的次数记录下来等。

### 静态代理、动态代理
代理的实现分为静态代理、动态代理：
* 静态代理：由开发人员创建或工具生成代理类的源码，再编译代理类。所谓静态也就是在程序运行前就已经存在代理类的字节码文件，代理类和委托类的关系在运行前就确定了。
* 动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。

静态代理：
```java
/**
 * 接口
 */
interface ITarget {
    void operation();
}

/**
 * 目标类
 */
class Target implements ITarget {

    @Override
    public void operation() {
        System.out.println("天气真好");
    }
}

/**
 * 代理类
 */
class Proxy implements ITarget {
    private ITarget target;

    public Proxy() {
        this.target = new Target();
    }

    @Override
    public void operation() {
        System.out.println("记录日志：开始调用目标方法Target.operation");
        long start = System.currentTimeMillis();
        target.operation();
        System.out.println("记录日志：目标方法Target.operation调用结束。耗时：" + (System.currentTimeMillis() - start));
    }
}

public class Main {
    public static void main(String[] args) {
        ITarget target = new Proxy();
        target.operation();
    }
}
```

JDK动态代理：
```java

/**
 * 接口
 */
interface ITarget {
    void operation();
}

/**
 * 目标类
 */
class Target implements ITarget {

    @Override
    public void operation() {
        System.out.println("天气真好");
    }
}

/**
 * 代理类
 */
class Proxy implements InvocationHandler {
    private Object target;

    /**
     * 绑定创建代理对象
     * @param target
     * @return
     */
    public Object bind(Object target) {
        this.target = target;
        return java.lang.reflect.Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("记录日志：开始调用目标方法Target.operation");
        long start = System.currentTimeMillis();
        Object invoke = method.invoke(target, args);
        System.out.println("记录日志：目标方法Target.operation调用结束。耗时：" + (System.currentTimeMillis() - start));
        return invoke;
    }
}

public class Main {
    public static void main(String[] args) {
        ITarget target = (ITarget) new Proxy().bind(new Target());
        target.operation();
    }
}
```
### JDK中代理模式使用
java.lang.reflect.Proxy：生成代理对象
RMI：代理远程调用

### 代理模式总结
适配器模式、装饰模式、代理模式三者之间看起来非常相似，之前在装饰模式中总结过适配器模式和装饰模式之前区别。对于装饰模式和代理模式区别在与：
* 1.对于静态代理，代理类是在编译的时候就确定了代理对象，而适配器是在调用的时候确定的被装饰对象。因为在使用装饰器的时候，通常是传递一个被装饰对象到装饰器中，而且装饰模式可以嵌套装饰。
* 2.对于动态代理，代理类虽然和装饰模式一样需要传递一个对象，但是代理类并不需要实现与目标类相同的接口。
动态代理UML类图：
![](http://otxnth5wx.bkt.clouddn.com/20171018屏幕快照2017-10-18上午11.39.05.png)

**使用场景：**
* 当客户端对象需要访问远程主机中的对象时可以使用远程代理。
* 当需要控制对一个对象的访问。
* 当需要为一个对象的访问（引用）提供一些额外的操作

**优点：**
* 能够协调调用者和被调用者，在一定程度上降低了系统的耦合度。
* 客户端可以针对抽象主题角色进行编程，增加和更换代理类无须修改源代码，符合开闭原则，系统具有较好的灵活性和可扩展性。
* 远程代理为位于两个不同地址空间对象的访问提供了一种实现机制，可以将一些消耗资源较多的对象和操作移至性能更好的计算机上，提高系统的整体运行效率。
* 还有其他不同代理类型优点不同

**缺点：**
* 由于在客户端和真实主题之间增加了代理对象，因此有些类型的代理模式可能会造成请求的处理速度变慢，例如保护代理。
* 实现代理模式需要额外的工作，而且有些代理模式的实现过程较为复杂，例如远程代理。
