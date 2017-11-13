---
title: Java设计模式-访问者模式（Visitor Pattern）
date: 2017-11-09 19:01:21
categories: ['设计模式']
tags: ['访问者模式', '设计模式', 'Visitor Pattern']
---

### 访问者模式
> 提供一个作用于某对象结构中的各元素的操作表示，它使我们可以在不改变各元素的类的前提下定义作用于这些元素的新操作。访问者模式是一种对象行为型模式。

访问者模式是一种较为复杂的行为型设计模式，它包含访问者和被访问元素两个主要组成部分，这些被访问的元素通常具有不同的类型，且不同的访问者可以对它们进行不同的访问操作。

访问者模式的目的是封装一些施加于某种数据结构元素之上的操作。一旦这些操作需要修改的话，接受这个操作的数据结构则可以保持不变。

UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171113屏幕快照2017-11-10上午9.55.04.png)
```java
import java.util.List;

/**
 * 抽象观察者
 */
abstract class AbstractVisitor {
    //观察子类A
    public abstract void visit(ConcreteElementA element);
    //观察子类B
    public abstract void visit(ConcreteElementB element);

    public void operation(){
        System.out.println("其他操作");
    }
}

/**
 * 实际观察者
 */
class ConcreteVisitor extends AbstractVisitor{

    @Override
    public void visit(ConcreteElementA element) {
        System.out.println("处理相关操作");
    }

    @Override
    public void visit(ConcreteElementB element) {
        System.out.println("处理相关操作");
    }
}

/**
 * 抽象元素
 */
abstract class Element {
    public List<Element> elements;

    public Element(List<Element> elements) {
        this.elements = elements;
    }

    public List<Element> getElements() {
        return elements;
    }

    public abstract void accept(AbstractVisitor visitor);
}

/**
 * 实体
 */
class ConcreteElementA extends Element {

    public ConcreteElementA(List<Element> elements) {
        super(elements);
    }

    @Override
    public void accept(AbstractVisitor visitor) {
        System.out.println("相关操作");
        visitor.visit(this);
        System.out.println("相关操作");
    }
}

/**
 * 实体
 */
class ConcreteElementB extends Element{

    public ConcreteElementB(List<Element> elements) {
        super(elements);
    }

    @Override
    public void accept(AbstractVisitor visitor) {
        System.out.println("相关操作");
        visitor.visit(this);
        System.out.println("相关操作");
    }
}
```
* AbstractVisitor（抽象访问者）：抽象访问者为对象结构中每一个具体元素类 ConcreteElement 声明一个访问操作，从这个操作的名称或参数类型可以清楚知道需要访问的具体元素的类型，具体访问者需要实现这些操作方法，定义对这些元素的访问操作。
* ConcreteVisitor（具体访问者）：具体访问者实现了每个由抽象访问者声明的操作，每一个操作用于访问对象结构中一种类型的元素。
* Element（抽象元素）：抽象元素一般是抽象类或者接口，它定义一个 accept() 方法，该方法通常以一个抽象访问者作为参数。
* ConcreteElement（具体元素）：具体元素实现了 accept() 方法，在 accept() 方法中调用访问者的访问方法以便完成对一个元素的操作。

访问者模式中对象结构存储了不同类型的元素对象，以供不同访问者访问。访问者模式包括两个层次结构，一个是访问者层次结构，提供了抽象访问者和具体访问者，一个是元素层次结构，提供了抽象元素和具体元素。相同的访问者可以以不同的方式访问不同的元素，相同的元素可以接受不同访问者以不同访问方式访问。在访问者模式中，增加新的访问者无须修改原有系统，系统具有较好的可扩展性。

### 实际举例
在一个公司中有正式员工和兼职员工，计算工资方式不一样。公司部门有财务部门和人力资源部门，财务部门查看统计员工工资，不同员工工资内容不同。人力资源部门查看统计员工工时。
UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171113屏幕快照2017-11-13下午2.39.10.png)
```java
/**
 * 部门
 */
interface Department {
    //计算
    void visit(RegularEmployee employee);

    void visit(PartTimeEmployee employee);
}

/**
 * 财务部门
 */
class FaDepartment implements Department {

    //计算工资
    @Override
    public void visit(RegularEmployee employee) {
        Integer workTime = employee.getWorkTime();
        Integer wages = employee.getMonthWages();
        if (workTime > 174) {
            wages = wages + (workTime - 174) * 10;
        } else if (workTime < 174) {
            wages = wages - (174 - workTime) * 20;
        }
        employee.setWages(wages);
        System.out.println("员工：" + employee.getName() + " 工资结算 " + wages);
    }

    @Override
    public void visit(PartTimeEmployee employee) {
        int wages = employee.getHourWages() * employee.getWorkTime();
        employee.setWages(wages);
        System.out.println("员工：" + employee.getName() + " 工资结算 " + wages);
    }
}

/**
 * HR部门
 */
class HRDepartment implements Department {

    @Override
    public void visit(RegularEmployee employee) {
        Integer workTime = employee.getWorkTime();
        System.out.println("员工：" + employee.getName() + "工作时间 " + employee.getWorkTime());
        if (workTime < 174) {
            System.out.println("员工：" + employee.getName() + " 本月有请假");
        }

    }

    @Override
    public void visit(PartTimeEmployee employee) {
        System.out.println("员工：" + employee.getName() + "工作时间 " + employee.getWorkTime());
    }
}

/**
 * 公司员工
 */
abstract class AbstractEmployee {
    private String name;

    //结算工资
    private Integer wages;

    public AbstractEmployee(String name) {
        this.name = name;
    }

    abstract void accept(Department department);

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getWages() {
        return wages;
    }

    public void setWages(Integer wages) {
        this.wages = wages;
    }
}

/**
 * 正式员工
 */
class RegularEmployee extends AbstractEmployee {
    //工资月工资
    private Integer monthWages;

    //工作时间
    private Integer workTime;

    public RegularEmployee(String name) {
        super(name);
    }

    @Override
    public void accept(Department department) {
        department.visit(this);
    }

    public Integer getMonthWages() {
        return monthWages;
    }

    public void setMonthWages(Integer monthWages) {
        this.monthWages = monthWages;
    }

    public Integer getWorkTime() {
        return workTime;
    }

    public void setWorkTime(Integer workTime) {
        this.workTime = workTime;
    }
}

/**
 * 兼职员工
 */
class PartTimeEmployee extends AbstractEmployee {
    //小时工资
    private Integer hourWages;
    //工作小时
    private Integer workTime;

    public PartTimeEmployee(String name) {
        super(name);
    }

    @Override
    public void accept(Department department) {
        department.visit(this);
    }

    public Integer getHourWages() {
        return hourWages;
    }

    public void setHourWages(Integer hourWages) {
        this.hourWages = hourWages;
    }

    public Integer getWorkTime() {
        return workTime;
    }

    public void setWorkTime(Integer workTime) {
        this.workTime = workTime;
    }
}


public class Main {
    public static void main(String[] args) {
        List<AbstractEmployee> employeeList = new ArrayList<>();
        PartTimeEmployee partTimeEmployee1 = new PartTimeEmployee("兼职1");
        partTimeEmployee1.setHourWages(5);
        partTimeEmployee1.setWorkTime(80);
        PartTimeEmployee partTimeEmployee2 = new PartTimeEmployee("兼职2");
        partTimeEmployee2.setHourWages(8);
        partTimeEmployee2.setWorkTime(40);

        RegularEmployee regularEmployee1 = new RegularEmployee("正式1");
        regularEmployee1.setMonthWages(5000);
        regularEmployee1.setWorkTime(180);
        RegularEmployee regularEmployee2 = new RegularEmployee("正式2");
        regularEmployee2.setMonthWages(4000);
        regularEmployee2.setWorkTime(150);

        employeeList.add(partTimeEmployee1);
        employeeList.add(partTimeEmployee2);
        employeeList.add(regularEmployee1);
        employeeList.add(regularEmployee2);

        FaDepartment faDepartment = new FaDepartment();
        HRDepartment hrDepartment = new HRDepartment();
        //HR查看员工
        employeeList.forEach(employee -> employee.accept(hrDepartment));
        int total = 0;
        for (AbstractEmployee employee : employeeList) {
            employee.accept(faDepartment);
            total += employee.getWages();
        }
        System.out.println("总共支付：" + total);
    }
}
```
如果需要新增访问者者，无需修改源码。如果需要新增具体元素，则需要修改所有的访问者。

### 分派
根据对象的类型而对方法进行的选择，就是分派。分派分为静态分派、动态分派。
静态分派：发生在编译时期，分派根据静态类型信息发生。（重载）
动态分派：发生在运行时期，动态分派地置换掉某个方法。（重写）

上面例子为 **伪双层分派**，员工调用accept只有一个方法，依据入参数得知部门，通过部门反过来调用方法visit得知入参。

* 参考：https://www.cnblogs.com/java-my-life/archive/2012/06/14/2545381.html

### 访问者总结

访问者模式在实际使用中相对较少。在XML文档解析、编译器设计、复杂集合对象的处理等领域访问者设计模式得到一定的应用。如：javax.lang.model.element.Element 、 javax.lang.model.element.ElementVisitor javax.lang.model.element.Element 、 javax.lang.model.element.ElementVisitor


**适用场景：**
* 一个对象结构包含多种类型的对象，希望对这些对象实施一些依赖其具体类型的操作。在访问者中针对每一种具体的类型都提供了一个访问操作，不同类型的对象可以有不同的访问操作。
* 需要对一个对象结构中的对象进行很多不同的并且不相关的操作，而需要避免让这些操作“污染”这些对象的类，也不希望在增加新操作时修改这些类。访问者模式使得我们可以将相关的访问操作集中起来定义在访问者类中，对象结构可以被多个不同的访问者类所使用，将对象本身与对象的访问操作分离。
* 对象结构中对象对应的类很少改变，但经常需要在此对象结构上定义新的操作。

**优点：**
* 增加新的访问操作很方便。使用访问者模式，增加新的访问操作就意味着增加一个新的具体访问者类，实现简单，无须修改源代码，符合“开闭原则”。
* 将有关元素对象的访问行为集中到一个访问者对象中，而不是分散在一个个的元素类中。类的职责更加清晰，有利于对象结构中元素对象的复用，相同的对象结构可以供多个不同的访问者访问。
* 让用户能够在不修改现有元素类层次结构的情况下，定义作用于该层次结构的操作。

**缺点：**
* 增加新的元素类很困难。在访问者模式中，每增加一个新的元素类都意味着要在抽象访问者角色中增加一个新的抽象操作，并在每一个具体访问者类中增加相应的具体操作，这违背了“开闭原则”的要求。
* 破坏封装。访问者模式要求访问者对象访问并调用每一个元素对象的操作，这意味着元素对象有时候必须暴露一些自己的内部操作和内部状态，否则无法供访问者访问。
