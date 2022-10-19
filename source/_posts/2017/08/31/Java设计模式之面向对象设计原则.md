---
title: Java设计模式之面向对象设计原则
date: 2017-08-31 16:07:21
categories: ['设计模式']
tags: ['Java', '设计模式', '面向对象设计原则']
---

## Java设计模式
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8A%E5%8D%888.33.33.png)
### 面向对象设计原则：
> * 单一职责原则：一个类只负责一个功能领域中的相应职责
> * 开闭原则：软件实体应对扩展开放，而对修改关闭
> * 里式代换原则：所有引用基类对象的地方能够透明地使用其子类的对象
> * 依赖倒转原则：抽象不应该依赖于细节，细节应该依赖于抽象
> * 接口隔离原则：使用多个专门的接口，而不使用单一的总接口
> * 合成复用原则：尽量使用对象组合，而不是继承来达到复用的目的
> * 迪米特法则：一个软件实体应当尽可能少地与其他实体发生相互作用

#### 单一职责原则（Single Responsibility Principle, SRP）：
> 单一职责原则是最简单的面向对象设计原则，它用于控制类的粒度大小。一个类只负责一个功能领域中的相应职责。

<!-- more -->
**问题** ：在系统一个类负责了多个功能，展示数据，处理逻辑，查询数据库，连接数据库。当需求发生变化时，逻辑变更，可能会导致其他功能发生故障。如下：
```Java
public class ActivityService {
    public void showAllAcitivty(){
        queryActivity();
        //处理其他相关业务逻辑
    }
    public Connection getConnection(){
        //连接数据库
        return null;
    }
    public List<Object> queryActivity(){
        getConnection();
        //查询数据
        //封装数据
        return new ArrayList<>();
    }
}
```
UML类图如下：
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8A%E5%8D%8810.52.32.png)
这里改类承担太多功能，需要从新改进，如下：
```Java
public class ActivityService {
    public void showAllAcitivty(){
        new ActivityDao().queryActivity();
        //处理其他相关业务逻辑
    }


}
public class ActivityDao{
    public List<Object> queryActivity(){
        DBUtils.getConnection();
        //查询数据
        //封装数据
        return new ArrayList<>();
    }
}
public class DBUtils{
    public static Connection getConnection(){
        //连接数据库
        return null;
    }
}
```
UML类图如下：
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8A%E5%8D%8811.40.30.png)
**注意**：在使用单一职责原则时，在拆分功能时，不要拆分太细，以免出现太多类。

#### 开闭原则（Open-Closed Principle, OCP）：
> 一个软件实体应当对扩展开放，对修改关闭。即软件实体应尽量在不修改原有代码的情况下进行扩展。开闭原则是面向对象的可复用设计的第一块基石，它是最重要的面向对象设计原则。在开闭原则的定义中，软件实体可以指一个软件模块、一个由多个类组成的局部结构或一个独立的类。

问题：现在需要依据不同的情况画不同的图，有饼图、条形图，项目后期有需求增加，需要增加折线图，这样就需要重新修改代码。如下：
```Java
public class ChartDisplay {
    public static void main(String[] args) {
        display("bar");
        display("pie");
    }
    //画图
    public static void display(String type){
        //如果每次增加一种类型都需要修改源码，新增判断
        if ("bar".equals(type)){
            new BarChart().display();
        } else if ("pie".equals(type)){
            new PieChart().display();
        }
    }
}
public class BarChart {
    public void display(){
        System.out.println("画条形图");
    }
}
public class PieChart {
    public void display(){
        System.out.println("画饼图");
    }
}
```
UML类图如下：
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8A%E5%8D%8811.46.40.png)
这里不符合开闭原则，需要代码重构，如下：
```Java
//新增一个画画抽象类
public abstract class AbstractChart {
    public abstract void display();
}
//具体子类实现
public class BarChart extends AbstractChart{
    @Override
    public void display(){
        System.out.println("画条形图");
    }
}
//具体子类实现
public class PieChart extends AbstractChart{
    @Override
    public void display(){
        System.out.println("画饼图");
    }
}

public class ChartDisplay {
    AbstractChart chart;

    public void setChart(AbstractChart chart){
        this.chart = chart;
    }
    public void display(){
        chart.display();
    }

    public static void main(String[] args) {
        ChartDisplay chartDisplay = new ChartDisplay();
        //这里setChart可以通过读取配置文件获取
        chartDisplay.setChart(new BarChart());
        chartDisplay.display();
        //这里setChart可以通过读取配置文件获取
        chartDisplay.setChart(new PieChart());
        chartDisplay.display();
    }

}
```
UML类图如下：
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8B%E5%8D%8812.00.36.png)
在改进方法后，在添加新的画图，只要添加一个新子类就可以实现。

**注**：在修改xml和properties等配置文件时，因为无需重新编译源代码，即可认为是符合开闭原则。

#### 里氏代换原则（Liskov Substitution Principle, LSP）
> 如果对每一个类型为 T1的对象 o1，都有类型为 T2 的对象o2，使得以 T1定义的所有程序 P 在所有的对象 o1 都代换成 o2 时，程序 P 的行为没有发生变化，那么类型 T2 是类型 T1 的子类型。所有引用基类的地方必须能透明地使用其子类的对象。也可以理解为：子类可以扩展父类的功能，但不能改变父类原有的功能。

这个不举例子，在使用java继承的时候，如果不覆盖父类方法就符合里氏代换原则。在使用过程中，如果可以尽量不要覆盖父类方法，可以使用重载，添加一个比父类更加宽松的行参。或者重新定义一个新方法。或者使用抽象编程。
```java
public class Father {
    public void insert(ArrayList<String> params ){
        System.out.println(this.getClass().getSimpleName() + ".insert");
    }
    public void select(){
        System.out.println(this.getClass().getSimpleName() + ".select");
    }
}
public class Child extends Father {
    public void insert(List<String> params){
        System.out.println(this.getClass().getSimpleName() + ".insert");
    }

    public void select(){
          System.out.println(this.getClass().getSimpleName() + ".select");
    }
}
public class Excemple{
    //在传递子类过来时，会执行子类的方法，不符合该原则
    public void select(Father father){
        father.select();
    }
}
```
#### 依赖倒转原则（Dependence Inversion Principle, DIP）
> 高层模块不应该依赖低层模块，二者都应该依赖其抽象；抽象不应该依赖细节；细节应该依赖抽象。换言之，要针对接口编程，而不是针对实现编程。

例子可以看之前开闭原则中，ChartDisplay.setChart传入的是抽象类，而不是具体的某一个实现类。

#### 接口隔离原则（Interface Segregation Principle, ISP）
> 使用多个专门的接口，而不使用单一的总接口，即客户端不应该依赖那些它不需要的接口。根据接口隔离原则，当一个接口太大时，我们需要将它分割成一些更细小的接口，使用该接口的客户端仅需知道与之相关的方法即可。每一个接口应该承担一种相对独立的角色，不干不该干的事，该干的事都要干。

Spring中XmlWebApplication的继承关系图：
![](/images/old/20170831%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-08-31%E4%B8%8B%E5%8D%883.24.01.png)

很多接口拆分的比较细

#### 合成复用原则（Composite Reuse Principle, CRP）
> 尽量使用对象组合，而不是继承来达到复用的目的。合成复用原则就是在一个新的对象里通过关联关系（包括组合关系和聚合关系）来使用一些已有的对象，使之成为新对象的一部分；新对象通过委派调用已有对象的方法达到复用功能的目的。简言之：复用时要尽量使用组合/聚合关系（关联关系），少用继承。

#### 迪米特法则（Law of Demeter, LOD）
> 如果一个系统符合迪米特法则，那么当其中某一个模块发生修改时，就会尽量少地影响其他模块，扩展会相对容易，这是对软件实体之间通信的限制，迪米特法则要求限制软件实体之间通信的宽度和深度。迪米特法则可降低系统的耦合度，使类与类之间保持松散的耦合关系。

当一个类与其他类之间依赖关系越多，耦合度越大时，类发生变化将会导致影响面越大、越广。而遵循迪米特法则就是为了尽量降低类与类之间的耦合度。迪米特法则定义不要和“陌生人”说话，只与你的直接朋友通信联系。朋友有：对象本身、类方法的（输入输出）参数、类成员。而出现在局部变量中的类就不是直接的朋友，即“陌生人”。在设计系统的时候，必须无法解耦，可以借用第三方进行转发调用（可使用外观模式Facade）。
