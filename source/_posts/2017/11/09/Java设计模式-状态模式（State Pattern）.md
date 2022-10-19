---
title: Java设计模式-状态模式（State Pattern）
date: 2017-11-09 10:55:21
categories: ['设计模式']
tags: ['设计模式', '状态模式', 'State Pattern']
---

### 状态设计模式
> 允许一个对象在其内部状态改变时改变它的行为，对象看起来似乎修改了它的类。其别名为状态对象(Objects for States)，状态模式是一种对象行为型模式。

状态模式用于解决系统中复杂对象的状态转换以及不同状态下行为的封装问题。当系统中某个对象存在多个状态，这些状态之间可以进行转换，而且对象在不同状态下行为不相同时可以使用状态模式。状态模式将一个对象的状态从该对象中分离出来，封装到专门的状态类中，使得对象状态可以灵活变化，对于客户端而言，无须关心对象状态的转换以及对象所处的当前状态，无论对于何种状态的对象，客户端都可以一致处理。

UML类图如下：
![](/images/old/20171109屏幕快照2017-11-09上午11.07.00.png)
<!-- more -->
```java
/**
 * 上下文环境类
 */
class Context{
    private AbstractState state;

    //状态变更后，环境方法
    public void request(){
        state.handle();
    }

    public AbstractState getState() {
        return state;
    }

    public void setState(AbstractState state) {
        this.state = state;
    }
}

/**
 * 抽象状态
 */
abstract class AbstractState{
    //抽象方法
    public abstract void handle();
}

/**
 * 具体状态
 */
class ConcreteStateA extends AbstractState{

    @Override
    public void handle() {

    }
}

/**
 * 具体状态
 */
class ConcreteStateB extends AbstractState{

    @Override
    public void handle() {

    }
}
```

* Context（环境类）：环境类又称为上下文类，它是拥有多种状态的对象。由于环境类的状态存在多样性且在不同状态下对象的行为有所不同，因此将状态独立出去形成单独的状态类。在环境类中维护一个抽象状态类State的实例，这个实例定义当前状态，在具体实现时，它是一个State子类的对象。
* AbstractState（抽象状态类）：它用于定义一个接口以封装与环境类的一个特定状态相关的行为，在抽象状态类中声明了各种不同状态对应的方法，而在其子类中实现类这些方法，由于不同状态下对象的行为可能不同，因此在不同子类中方法的实现可能存在不同，相同的方法可以写在抽象状态类中。
* ConcreteState（具体状态类）：它是抽象状态类的子类，每一个子类实现一个与环境类的一个状态相关的行为，每一个具体状态类对应环境的一个具体状态，不同的具体状态类其行为有所不同。

### 实际例子
现在有一款产品在上架前需要审核等操作，产品有多种状态。设计如下
```java
/**
 * 产品上下文环境类
 */
class Product{
    private AbstractState state;

    //状态变更后，环境方法
    public void request(){
        state.handle();
    }

    public AbstractState getState() {
        return state;
    }

    public void setState(AbstractState state) {
        this.state = state;
    }
}

/**
 * 状态接口
 */
interface AbstractState{
    void handle();
}

/**
 * 具体状态
 */
enum ProductState implements AbstractState{
    //草稿状态
    DRAFT {
        @Override
        public void handle() {
            System.out.println("保存草稿！");
        }
    },
    //提交审核
    REVIEW{
        @Override
        public void handle() {
            System.out.println("保存审核状态");
            System.out.println("通知审核人员");
        }
    },
    //审核通过
    SUCCESS{
        @Override
        public void handle() {
            System.out.println("审核通过，产品上架");
            System.out.println("通知商家");
        }
    },
    //审核失败
    FAIL{
        @Override
        public void handle() {
            System.out.println("审核失败通知商家");
        }
    }

}

public class Main {
    public static void main(String[] args)  {
        Product product = new Product();
        //设置
        product.setState(ProductState.DRAFT);
        //保存草稿
        product.request();
        //提交审核
        product.setState(ProductState.REVIEW);
        product.request();

    }
}
```
这里状态采用的是接口，实际产品状态为枚举，产品本身类和状态实现分离。各自关注自己业务。也可以吧状态转换放入环境类，通过触发不同的条件，改变当前状态。

### 共享状态
在有些情况下，多个环境对象可能需要共享同一个状态，如果希望在系统中实现多个环境对象共享一个或多个状态对象，那么需要将这些状态对象定义为环境类的静态成员对象。
原理就是把状态实在为静态属性。所有对象共享。

### 状态模式总结
状态模式将一个对象在不同状态下的不同行为封装在一个个状态类中，通过设置不同的状态对象可以让环境对象拥有不同的行为，而状态转换的细节对于客户端而言是透明的，方便了客户端的使用。在实际开发中，状态模式具有较高的使用频率，在工作流和游戏开发中状态模式都得到了广泛的应用，例如公文状态的转换、游戏中角色的升级等。

**适用场景：**
* 对象的行为依赖于它的状态，状态的改变将导致行为的变化。
* 代码中有大量于对象有个的条件语句。

**优点：**
* 如果在环境类中封装状态改变，可以对状态进行集中管理，而不是分散在业务中。
* 将所有与某个状态有关的行为放到一个类中，只需要注入一个不同的状态对象即可使环境对象拥有不同的行为。
* 允许状态转换逻辑与状态对象合成一体，而不是提供一个巨大的条件语句块，状态模式可以让我们避免使用庞大的条件语句来将业务方法和状态转换代码交织在一起。
* 可以让多个环境对象共享一个状态对象，从而减少系统中对象的个数。

**缺点：**
* 状态模式的使用必然会增加系统中类和对象的个数，导致系统运行开销增大。
* 状态模式的结构与实现都较为复杂，如果使用不当将导致程序结构和代码的混乱，增加系统设计的难度。
* 状态模式对“开闭原则”的支持并不太好，增加新的状态类需要修改那些负责状态转换的源代码，否则无法转换到新增状态；而且修改某个状态类的行为也需修改对应类的源代码。
