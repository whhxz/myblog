---
title: Java设计模式-观察者模式（Observer Pattern）
date: 2017-11-02 10:32:21
categories: ['设计模式']
tags: ['设计模式', '观察者模式', 'Observer Pattern']
---

### 观察者模式
> 定义对象之间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。观察者模式的别名包括发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。观察者模式是一种对象行为型模式。

观察者模式是使用频率最高的设计模式之一，它用于建立一种对象与对象之间的依赖关系，一个对象发生改变时将自动通知其他对象，其他对象将相应作出反应。在观察者模式中，发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间可以没有任何相互联系，可以根据需要增加和删除观察者，使得系统更易于扩展。

在路上，交通信号灯属于观察者模式，驾驶员观察信号灯的变化，做相应的动作。

UML类图如下：
![](/images/old/20171104屏幕快照2017-11-02上午10.58.18.png)
<!-- more -->
```java
/**
 * 目标类
 */
abstract class AbstractSubject {
    //定义观察者集合存储观察者，可抽取到第三方类
    protected List<Observer> observerList = new ArrayList<>();

    /**
     * 新增观察者
     * @param observer
     */
    public void attach(Observer observer){
        observerList.add(observer);
    }

    /**
     * 删除观察者
     * @param observer
     */
    public void detach(Observer observer){
        observerList.remove(observer);
    }

    public abstract void notifyObserver();
}

/**
 * 具体目标类
 */
class ConcreteSubject extends AbstractSubject {

    @Override
    public void notifyObserver() {
        System.out.println("通知前操作");
        for (Observer observer : observerList) {
            //一般避免阻塞，此处可设置为异步
            observer.update();
        }
        System.out.println("通知完毕");
    }
}

/**
 * 观察者
 */
interface Observer{
    void update();
}

/**
 * 具体观察者
 */
class ConcreteObserver implements Observer{

    @Override
    public void update() {
        System.out.println("通过更新");
    }
}
```
* Subject（目标）：目标又称为主题，它是指被观察的对象。在目标中定义了一个观察者集合，一个观察目标可以接受任意数量的观察者来观察，它提供一系列方法来增加和删除观察者对象，同时它定义了通知方法。
* ConcreteSubject（具体目标）：具体目标是目标类的子类，通常它包含有经常发生改变的数据，当它的状态发生改变时，向它的各个观察者发出通知；
* Observer（观察者）：观察者将对观察目标的改变做出反应，观察者一般定义为接口，该接口声明了更新数据的方法update()，因此又称为抽象观察者。
* ConcreteObserver（具体观察者）：在具体观察者中维护一个指向具体目标对象的引用，它存储具体观察者的有关状态，这些状态需要和具体目标的状态保持一致；通常在实现时，可以调用具体目标类的attach()方法将自己添加到目标类的集合中或通过detach()方法将自己从目标类的集合中删除。

在实际开发中，可能相对复杂，在目标类通知观察者时，可能会依据需求做相应调整，如观察者入参添加目标类，或者直接添加目标类相应属性。

举例：
在用户购买商品后，可能需要记录日志、记录订单、通知大数据、通知商家、通知客服系统、通知活动、通知优惠卷系统等。如果在开发中，增加或者修改其中的业务，会因为系统耦合度太高修改困难。可以采用观察者模式，解决系统之间耦合太高。
具体代码略。

#### Java中使用
```java
java.util.Observer
java.util.EventListener
javax.servlet.http.HttpSessionBindingListener
```

Observer由jdk提供的观察者。对于不是很复杂需求，可以直接使用。
在阅读spring源码时，有不少fireXXXX()方法，这是在事件触发后调用。
### 类似设计模式比较
在学习设计模式过程中，理解不同的设计模式时，有时候困惑设计模式之间的差异

**外观设计模式：** 客户端类与多个其他业务类交互，如果平常开发，是一个类里面有其他业务类的引用。如果现在有另外一个客户端需要需要与这些业务相交互，也需要同时引入这些类，这个时候需要引入一个外观类来减少客户端对外部其他业务类的耦合，由外观类关联其他业务系统。对于外观设计模式而言不涉及到具体行为，只是类与类之间的组合关系，所以外观类属于结构型设计模式。
举例：现在A想去新马泰旅游（不需要管具体细节如坐车、买票等行为），A需要了解新加坡（持有新加坡对象）、马来西亚（持有马来西亚对象）、泰国（持有泰国对象），如果需要新增地点，需要同时了解新地点以及持有新地点，过于繁琐。现在因为外部因素，时间精力不够等，需要解决这种问题，可以找旅行社定制旅游计划，A用户只需要和旅行社联系即可，想去哪里直接和旅行社打交道，不需要与具体地点联系。与此同时如果有B也有同样的需要，就可以减少类之间的交互。

**代理模式：** 客户端无法直接访问某个对象，或者访问对象困难，这个时候需要引入代理对象。因为客户端和实际对象之间存在代理人，代理人访问目标类时，可以做一些额外操作。代理模式因为不涉及到具体操作只是对象之间关系，所以属于代理模式。
举例：接上述旅游例子，A出去旅游，A朋友希望A能帮忙代购一些东西，可能因为国内物品较贵或者国内没有，这个时候A属于代理对象，A朋友属于客户端，代购物品属于目标类。

**中介者模式：** 系统对象之间需要互相联系，导致整个系统呈网状结构，系统直接存在大量多对多联系，系统过于复杂。这个时候需要改变网状结构为星状结构，引入一个中介者，系统之间联系都通知中介者，由中介者通知其他系统。
举例：接上诉例子，A出去旅游其实并不是自己一个人，还有其他一起的朋友，A需要代购的朋友也不止一个，旅游的朋友也有代购，这个时候就会出现沟通的问题，为了避免大家浪费时间，A组建了一个代购旅游群，大家都在里面交流，交流旅游心得、代购物品等。这个代购群，就属于中介者。

**观察者模式：** 观察者模式用于一个对象发送改变后，通知其他对象。
举例：接上面例子，A因为出去旅游的次数多，每次都走的旅行社，成为了旅行社的VIP，加了旅行社的公众号，因为旅行社对于这种VIP用户是有优惠活动，有优惠的时候会通知导致VIP用户。这个时候如果就是观察者模式，A以及该旅行社VIP属于观察者（订阅者），旅行社有VIP活动（目标类变动）会通知到VIP用户。

### 观察者总结

观察者模式是一种使用频率非常高的设计模式，无论是移动应用、Web应用或者桌面应用，观察者模式几乎无处不在，它为实现对象之间的联动提供了一套完整的解决方案，凡是涉及到一对一或者一对多的对象交互场景都可以使用观察者模式。观察者模式广泛应用于各种编程语言的GUI事件处理的实现，在基于事件的XML解析技术（如SAX2）以及Web事件处理中也都使用了观察者模式。

**应用场景：**
* 一个抽象模型有两个方面，其中一个方面依赖于另一个方面，将这两个方面封装在独立的对象中使它们可以各自独立地改变和复用。（关联行为是可拆分）
* 一个对象的改变将导致一个或多个其他对象也发生改变，而并不知道具体有多少对象将发生改变，也不知道这些对象是谁。
* 需要在系统中创建一个触发链，A对象的行为将影响B对象，B对象的行为将影响C对象……，可以使用观察者模式创建一种链式触发机制。
* 跨系统的消息交换场景，如消息队列的处理机制。

**优点：**
* 观察者模式可以实现表示层和数据逻辑层的分离，定义了稳定的消息更新传递机制，并抽象了更新接口，使得可以有各种各样不同的表示层充当具体观察者角色。
* 观察者模式在观察目标和观察者之间建立一个抽象的耦合。观察目标只需要维持一个抽象观察者的集合，无须了解其具体观察者。由于观察目标和观察者没有紧密地耦合在一起，因此它们可以属于不同的抽象化层次。
* 观察者模式支持广播通信，观察目标会向所有已注册的观察者对象发送通知，简化了一对多系统设计的难度。
* 观察者模式满足“开闭原则”的要求，增加新的具体观察者无须修改原有系统代码，在具体观察者与观察目标之间不存在关联关系的情况下，增加新的观察目标也很方便。

**缺点：**
* 如果一个观察目标对象有很多直接和间接观察者，将所有的观察者都通知到会花费很多时间。
* 如果在观察者和观察目标之间存在循环依赖，观察目标会触发它们之间进行循环调用，可能导致系统崩溃。(互相观察)
* 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化。
