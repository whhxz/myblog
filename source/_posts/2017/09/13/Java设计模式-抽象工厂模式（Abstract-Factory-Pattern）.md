---
title: Java设计模式-抽象工厂模式（Abstract Factory Pattern）
date: 2017-09-13 10:48:25
categories: ['设计模式']
tags: ['设计模式', '抽象工厂', 'Abstract Factory Pattern']
---

### 抽象工厂概念
> 抽象工厂相对工厂方法，可以创建一个产品族，而不是单一的产品。也就是在工厂方法上添加了创建其他产品方法。

如图：
![](/images/old/20170913%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A72017-09-13%E4%B8%8B%E5%8D%882.17.10.png)
* SeasonFactory：季节工厂
* SpringFactory：春天工厂
* SummerFactory：夏天工厂
* Clothes：衣服
* Coat:上衣
* Trousers：裤子
Java代码如下：<!-- more -->
```java
interface Clothes{
    void coth();
}

interface Coat extends Clothes{

}

interface Trousers extends Clothes{
}

class SpringCoat implements Coat{

    @Override
    public void coth() {
        System.out.println("春天的上衣");
    }
}
class SummerCoat implements Coat{

    @Override
    public void coth() {
        System.out.println("夏天的上衣");
    }
}

class SpringTrousers implements Trousers{

    @Override
    public void coth() {
        System.out.println("春天的裤子");
    }
}
class SummerTrousers implements Trousers{

    @Override
    public void coth() {
        System.out.println("夏天的裤子");
    }
}



interface SeaconFactory{
    Clothes createCoat();
    Clothes createTrousers();
}

class SpringFactory implements SeaconFactory{

    @Override
    public Clothes createCoat() {
        Clothes coat = new SpringCoat();
        System.out.println("创建：" + coat.getClass().getSimpleName());
        //do some thing
        return coat;
    }

    @Override
    public Clothes createTrousers() {
        Clothes trousers = new SpringTrousers();
        System.out.println("创建：" + trousers.getClass().getSimpleName());
        //do some thing
        return trousers;

    }
}

class SummerFactory implements SeaconFactory{
    @Override
    public Clothes createCoat() {
        Clothes clothes = new SummerCoat();
        System.out.println("创建：" + clothes.getClass().getSimpleName());
        //do some thing
        return clothes;
    }

    @Override
    public Clothes createTrousers() {
        Clothes clothes = new SummerTrousers();
        System.out.println("创建：" + clothes.getClass().getSimpleName());
        //do some thing
        return clothes;
    }
}
```
在范例中，如果要创建夏天的一套衣服，直接使用SummerFactory创建上衣和裤子就可以，这样的话身上一套就是夏天的装扮，避免因为出现混搭而出现问题。在实际生活中，电脑主板和CPU的针脚是需要对应，可以使用该抽象方法，避免组别和CUP针脚对不上。
### 抽象工厂模式优点
1. 一个工厂车间一系列相关相互依赖的对象，避免出现混合。
2. 新增产品族时，无需修改系统，比如冬天的衣服。符合“开闭原则”。

### 抽象工厂模式缺点
1. 新增产品的时候比较麻烦，比如新增鞋子。
### 总结
抽象工厂是在工厂方法模式上的增强，一个工厂类可以创建多个产品。
