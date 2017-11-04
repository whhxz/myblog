---
title: Java设计模式-中介者模式（Mediator Pattern）
date: 2017-10-30 13:59:18
categories: ['设计模式']
tags: ['设计模式', '中介者模式', 'Mediator Pattern']
---

### 中介者模式
> 用一个中介对象（中介者）来封装一系列的对象交互，中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。中介者模式又称为调停者模式，它是一种对象行为型模式。

如果在一个系统中对象之间的联系呈现为网状结构，对象之间存在大量的多对多联系，将导致系统非常复杂，这些对象既会影响别的对象，也会被别的对象所影响，这些对象称为同事对象，它们之间通过彼此的相互作用实现系统的行为。在网状结构中，几乎每个对象都需要与其他对象发生相互作用，而这种相互作用表现为一个对象与另外一个对象的直接耦合，这将导致一个过度耦合的系统。中介者模式可以使对象之间的关系数量急剧减少，通过引入中介者对象，可以将系统的网状结构变成以中介者为中心的星形结构，“在这个星形结构中，同事对象不再直接与另一个对象联系，它通过中介者对象与另一个对象发生相互作用。中介者对象的存在保证了对象结构上的稳定，也就是说，系统的结构不会因为新对象的引入带来大量的修改工作。如果在一个系统中对象之间存在多对多的相互关系，我们可以将对象之间的一些交互行为从各个对象中分离出来，并集中封装在一个中介者对象中，并由该中介者进行统一协调，这样对象之间多对多的复杂关系就转化为相对简单的一对多关系。通过引入中介者来简化对象之间的复杂交互，中介者模式是“迪米特法则”的一个典型应用。
网状图：
![](http://otxnth5wx.bkt.clouddn.com/20171030屏幕快照2017-10-30下午2.34.34.png)
星状图：
![](http://otxnth5wx.bkt.clouddn.com/20171030屏幕快照2017-10-30下午2.34.41.png)

UML类图：
![](http://otxnth5wx.bkt.clouddn.com/20171030屏幕快照2017-10-30下午3.00.36.png)
```java
/**
 * 抽象中介者
 */
abstract class AbstractMediator{
    protected List<AbstractColleague> colleagues = new ArrayList<>();

    /**
     * 注册
     * @param colleague
     */
    public void register(AbstractColleague colleague){
        colleagues.add(colleague);
    }

    /**
     * 相关操作
     */
    public abstract void operation();
}

/**
 * 中介者实现类
 */
class ConcreteMediator extends AbstractMediator{
    @Override
    public void operation() {
        System.out.println("中介者开始通知");
        colleagues.forEach(AbstractColleague::callback);
    }
}

/**
 * 抽象同事类
 */
abstract class AbstractColleague{
    protected AbstractMediator mediator;

    public AbstractColleague(AbstractMediator mediator) {
        this.mediator = mediator;
    }

    /**
     * 消息回调
     */
    public abstract void callback();

    /**
     * 发送消息
     */
    public void contact(){
        mediator.operation();
    }
}

class ConcreteColleagueA extends AbstractColleague{

    public ConcreteColleagueA(AbstractMediator mediator) {
        super(mediator);
    }

    @Override
    public void callback() {
        System.out.println("A收到消息");
    }
}
class ConcreteColleagueB extends AbstractColleague{

    public ConcreteColleagueB(AbstractMediator mediator) {
        super(mediator);
    }

    @Override
    public void callback() {
        System.out.println("B收到消息");
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException, ScriptException {
        AbstractMediator mediator = new ConcreteMediator();

        ConcreteColleagueA colleagueA = new ConcreteColleagueA(mediator);
        mediator.register(colleagueA);
        ConcreteColleagueB colleagueB = new ConcreteColleagueB(mediator);
        mediator.register(colleagueB);

        colleagueA.contact();
    }
}
```
* Mediator（抽象中介者）：它定义一个接口，该接口用于与各同事对象之间进行通信。
* ConcreteMediator（具体中介者）：它是抽象中介者的子类，通过协调各个同事对象来实现协作行为，它维持了对各个同事对象的引用。
* Colleague（抽象同事类）：它定义各个同事类公有的方法，并声明了一些抽象方法来供子类实现，同时它维持了一个对抽象中介者类的引用，其子类可以通过该引用来与中介者通信。
* ConcreteColleague（具体同事类）：它是抽象同事类的子类；每一个同事对象在需要和其他同事对象通信时，先与中介者通信，通过中介者来间接完成与其他同事类的通信；在具体同事类中实现了在抽象同事类中声明的抽象方法。
* 同事类指互相之间有关联的类。

### 外观模式、代理模式、中介者模式
**外观模式：** 为子系统中的一组接口提供一个统一的入口。外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。
在外观模式中，外观模式所有的请求处理都委托给子系统完成，而中介者模式则由中心协调同事类和中心本身共同完成业务。外观模式属于单向操作，调用者和子系统可以理解为1->N。

**代理模式：** 给某一个对象提供一个代理或占位符，并由代理对象来控制对原对象的访问。在某些情况下，一个对象不适合或者不能直接引用另一个对象，而代理对象可以在客户端和目标对象之间起到传递的作用。
在代理模式中，代理类用于隐藏真实对象、扩展额外功能。可以理解为代理模式为1->1。

**中介者模式：** 可以理解为N<-->N解耦。

### 中介者模式总结
在中介者模式中，中介者承担两方面的职责：
* 中转作用（结构性）：通过中介者提供的中转作用，各个同事对象就不再需要显式引用其他同事，当需要和其他同事进行通信时，通过中介者即可。该中转作用属于中介者在结构上的支持。
* 协调作用（行为性）：中介者可以更进一步的对同事之间的关系进行封装，同事可以一致地和中介者进行交互，而不需要指明中介者需要具体怎么做，中介者
根据封装在自身内部的协调逻辑，对同事的请求进行进一步处理，将同事成员之间的关系行为进行分离和封装。该协调作用属于中介者在行为上的支持。

在群组聊天中，可以采用中介者模式。
一般在需要抽取一个可以管理其他业务的对象作为中介者，各个相关对象互相之间不打交道，与中介者打交道，之后由中介者通过相关规则通知到其他关联对象。
如果在系统中存在大量多对多关系时，不需要第一时间采用中介者模式，可以考虑下是否系统设计不够合理。

在JDK中应用中介者模式有：
* All scheduleXXX() methods of java.util.Timer
* java.util.concurrent.Executor#execute()
* submit() and invokeXXX() methods of java.util.concurrent.ExecutorService
* scheduleXXX() methods of java.util.concurrent.ScheduledExecutorService
* java.lang.reflect.Method#invoke()
由于能力有限，并未找到上述类中使用了中介者模式，也未找到相关文档。

**适用场景：**
* 系统中对象之间存在复杂的引用关系，产生的相互依赖关系结构混乱且难以理解。
* 一个对象由于引用了其他很多对象并且直接和这些对象通信，导致难以复用该对象。
* 想通过一个中间类来封装多个类中的行为，而又不想生成太多的子类。可以通过引入中介者类来实现，在中介者中定义对象。
* 交互的公共行为，如果需要改变行为则可以增加新的中介者类。

**优点：**
* 中介者模式简化了对象之间的交互，它用中介者和同事的一对多交互代替了原来同事之间的多对多交互，一对多关系更容易理解、维护和扩展，将原本难以理解的网状结构转换成相对简单的星型结构。
* 中介者模式可将各同事对象解耦。中介者有利于各同事之间的松耦合，我们可以独立的改变和复用每一个同事和中介者，增加新的中介者和新的同事类都比较方便，更好地符合“开闭原则”。
* 可以减少子类生成，中介者将原本分布于多个对象间的行为集中在一起，改变这些行为只需生成新的中介者子类即可，这使各个同事类可被重用，无须对同事类进行扩展。

**缺点：**
* 在具体中介者类中包含了大量同事之间的交互细节，可能会导致具体中介者类非常复杂，使得系统难以维护。
