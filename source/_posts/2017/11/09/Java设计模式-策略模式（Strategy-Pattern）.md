---
title: Java设计模式-策略模式（Strategy Pattern）
date: 2017-11-09 15:57:30
categories: ['设计模式']
tags: ['策略模式', '设计模式', 'Strategy Pattern']
---

### 策略模式
> 定义一系列算法类，将每一个算法封装起来，并让它们可以相互替换，策略模式让算法独立于使用它的客户而变化，也称为政策模式(Policy)。策略模式是一种对象行为型模式。

在策略模式中，我们可以定义一些独立的类来封装不同的算法，每一个类封装一种具体的算法，在这里，每一个封装算法的类我们都可以称之为一种策略(Strategy)，为了保证这些策略在使用时具有一致性，一般会提供一个抽象的策略类来做规则的定义，而每种算法则对应于一个具体策略类。

UML类图如下：
![](http://image.whhxz.smallstool.cn/20171109屏幕快照2017-11-09下午4.14.50.png)
<!-- more -->
```java

/**
 * 环境类
 */
class Context {
    public void algorithm(AbstractStrategy strategy) {
        //执行算法
        strategy.algorithm();
    }
}


/**
 * 抽象策略
 */
abstract class AbstractStrategy {
    //算法
    abstract void algorithm();
}

/**
 * 算法A
 */
class ConcreteStrategyA extends AbstractStrategy {

    @Override
    void algorithm() {
        System.out.println("算法A");
    }
}

/**
 * 算法B
 */
class ConcreteStrategyB extends AbstractStrategy {

    @Override
    void algorithm() {
        System.out.println("算法B");
    }
}
```

* Context（环境类）：环境类是使用算法的角色，它在解决某个问题（即实现某个方法）时可以采用多种策略。
* AbstractStrategy（抽象策略类）：它为所支持的算法声明了抽象方法，是所有策略类的父类，它可以是抽象类或具体类，也可以是接口。环境类通过抽象策略类中声明的方法在运行时调用具体策略类中实现的算法。
* ConcreteStrategy（具体策略类）：它实现了在抽象策略类中声明的算法，在运行时，具体策略类将覆盖在环境类中定义的抽象策略类对象，使用一种具体的算法实现某个业务处理。

这里策略模式为了方便连接，在Context类中没有策略属性，只是Context中方法有策略的入参数，不这样做的原因是为了避免和状态模式弄混，具体实际开发可能并不是单一的设计模式，可以混合着用。

### 实际例子
现在需要通过购物车中商品扣减活动优惠，得到实际订单价格。
```java
import java.util.List;

/**
 * 购物车
 */
class ShoppingCart {
    //购物车商品
    private List<String> products;
    //购物车价格
    private Integer price;

    public void joinActivity(AbstractActivity activity) {
        //执行计算
        activity.algorithm(this);
    }

    public Integer getPrice() {
        return price;
    }

    public void setPrice(Integer price) {
        this.price = price;
    }
}


/**
 * 抽象策略
 */
abstract class AbstractActivity {
    //算法
    abstract void algorithm(ShoppingCart cart);
}

/**
 * 优惠活动A
 */
class ConcreteActivityA extends AbstractActivity {


    @Override
    void algorithm(ShoppingCart cart) {
        System.out.println("优惠活动A：计算购物车价格");
        cart.setPrice(cart.getPrice() - 10);
    }
}

/**
 * 优惠活动B
 */
class ConcreteActivityB extends AbstractActivity {

    @Override
    void algorithm(ShoppingCart cart) {
        System.out.println("优惠活动B：计算购物车价格");
        cart.setPrice(cart.getPrice() - 20);
    }
}

public class Main {
    public static void main(String[] args) {
        ShoppingCart shoppingCart = new ShoppingCart();
        //购物车总价100
        shoppingCart.setPrice(100);
        //可以从其他地方获取购物车中商品参加的活动
        shoppingCart.joinActivity(new ConcreteActivityA());
        System.out.println(shoppingCart.getPrice());

    }
}
```
例子比较简单，如果设置不同的活动，对购物车的商品进行扣减。

### JDK中应用
![](http://image.whhxz.smallstool.cn/20171109屏幕快照2017-11-09下午4.59.53.png)
图中少了一根Container指向LayoutManager的线，Container中有个LayoutManager属性，在使用Container的时候，通过设置不同Layout展示不同布局。

### 策略模式总结
在上面策略模式UML图中，如果把策略类设置为环境类的属性，那么策略模式和状态模式的UML类图是一模一样的。
如果在环境类中有多个方法，几乎每个方法都与当前某个属性有关，那么可以采用状态模式，如果只是一个方法有关，或者相关很少，可以采用策略模式。依据实际需求设计。
可参考：[https://www.zhihu.com/question/23693088](https://www.zhihu.com/question/23693088)

**适用场景：**
* 一个系统需要动态地在几种算法中选择一种，那么可以将这些算法封装到一个个的具体算法类中，而这些具体算法类都是一个抽象算法类的子类。
* 一个对象有很多的行为，如果不用恰当的模式，这些行为就只好使用多重条件选择语句来实现。此时，使用策略模式，把这些行为转移到相应的具体策略类里面，就可以避免使用难以维护的多重条件选择语句。
* 不希望客户端知道复杂的、与算法相关的数据结构，在具体策略类中封装算法与相关的数据结构，可以提高算法的保密性与安全性。

**优点：**
* 策略模式提供了对“开闭原则”的完美支持，用户可以在不修改原有系统的基础上选择算法或行为，也可以灵活地增加新的算法或行为。
* 策略模式提供了管理相关的算法族的办法。策略类的等级结构定义了一个算法或行为族，恰当使用继承可以把公共的代码移到抽象策略类中，从而避免重复的代码。
* 策略模式提供了一种算法的复用机制，由于将算法单独提取出来封装在策略类中，因此不同的环境类可以方便地复用这些策略类。

**缺点：**
* 客户端必须知道所有的策略类，并自行决定使用哪一个策略类。这就意味着客户端必须理解这些算法的区别，以便适时选择恰当的算法。换言之，策略模式只适用于客户端知道所有的算法或行为的情况。
* 策略模式将造成系统产生很多具体策略类，任何细小的变化都将导致系统要增加一个新的具体策略类。
* 无法同时在客户端使用多个策略类，也就是说，在使用策略模式时，客户端每次只能使用一个策略类，不支持使用一个策略类完成部分功能后再使用另一个策略类来完成剩余功能的情况。
