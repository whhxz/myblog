---
title: Java设计模式-桥接模式（Bridge Pattern）
date: 2017-09-20 19:16:12
categories: ['设计模式']
tags: ['设计模式', '桥接模式', 'Bridge Pattern']
---

### 处理多维度变化--桥接模式
> 将抽象部分与它的实现部分分离，使它们都可以独立地变化。它是一种对象结构型模式，又称为柄体(Handle and Body)模式或接口(Interface)模式。

桥接模式用一种巧妙的方式处理多层继承存在的问题，用抽象关联取代了传统的多层继承，将类之间的静态继承关系转换为动态的对象组合关系，使得系统更加灵活，并易于扩展，同时有效控制了系统中类的个数。

错误示范：
要设计武器，要有基本武器，还有附魔武器。
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午7.49.52.png)
<!-- more -->
采用桥接模式：
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午8.06.56.png)
```Java
**
 * 武器
 */
interface Weapon{
    /**
     * 轻攻击
     */
    void lightAttack();

    /**
     * 重攻击
     */
    void heavyAttack();
}

/**
 * 附魔
 */
interface Enchantment{
    /**
     * 激活
     */
    void onActivate();

    /**
     * 使用
     */
    void apply();

    /**
     * 失效
     */
    void onDeactivate();
}

/**
 * 出血附魔
 */
class BleedingEnchantment implements Enchantment{

    @Override
    public void onActivate() {
        System.out.println("激活出血效果");
    }

    @Override
    public void apply() {
        System.out.println("出血伤害");
    }

    @Override
    public void onDeactivate() {
        System.out.println("取消出血效果");
    }
}

/**
 * 暴击附魔
 */
class StrikeEnchantment implements Enchantment{
    @Override
    public void onActivate() {
        System.out.println("激活暴击效果");
    }

    @Override
    public void apply() {
        System.out.println("暴击伤害");
    }

    @Override
    public void onDeactivate() {
        System.out.println("取消暴击效果");
    }
}

/**
 * 剑
 */
class Sword implements Weapon{
    //附魔
    private Enchantment enchantment;

    public Sword(Enchantment enchantment) {
        this.enchantment = enchantment;
    }

    @Override
    public void lightAttack() {
        if (enchantment != null){
            enchantment.onActivate();
            enchantment.apply();
        }
        System.out.println("剑轻攻击");
        if (enchantment != null){
            enchantment.onDeactivate();
        }
    }

    @Override
    public void heavyAttack() {
        if (enchantment != null){
            enchantment.onActivate();
            enchantment.apply();
        }
        System.out.println("剑重攻击");
        if (enchantment != null){
            enchantment.onDeactivate();
        }
    }
}

/**
 * 大锤
 */
class Hammer implements Weapon{
    //附魔
    private Enchantment enchantment;

    public Hammer(Enchantment enchantment) {
        this.enchantment = enchantment;
    }

    @Override
    public void lightAttack() {
        if (enchantment != null){
            enchantment.onActivate();
            enchantment.apply();
        }
        System.out.println("大锤轻攻击");
        if (enchantment != null){
            enchantment.onDeactivate();
        }
    }

    @Override
    public void heavyAttack() {
        if (enchantment != null){
            enchantment.onActivate();
            enchantment.apply();
        }
        System.out.println("大锤重攻击");
        if (enchantment != null){
            enchantment.onDeactivate();
        }
    }
}

public class Main {
    public static void main(String[] args) {
        Enchantment strikeEnchantment = new StrikeEnchantment();
        Weapon hammer = new Hammer(strikeEnchantment);
        hammer.heavyAttack();

        Enchantment bleedingEnchantment = new BleedingEnchantment();
        Weapon sword = new Sword(bleedingEnchantment);
        sword.lightAttack();
    }
}
```
### Java中使用的桥接
java.util.logging.Handler中Formatter，采用的是桥接设计模式。
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午8.49.05.png)
Handler子类有ConsoleHandler、FileHandler、SocketHandler等
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午8.52.39.png)
Formatter子类有XMLFormatter、SimpleFormatter等


在Handler子类中有实现publish中，有需要使用Formatter来格式化输出，如图
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午8.55.23.png)

JDBC连接核心也是使用的桥接模式，如下：
Class.forName("com.mysql.jdbc.Driver");
DriverManager.getConnection("url");
在com.mysql.jdbc.Driver类加载的时候有个静态方法，把当前对象加载到DriverManager.registeredDrivers中
在使用getConnection的时候，获取到之前的驱动连接数据库。如图：
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午9.05.37.png)
![](http://image.whhxz.smallstool.cn/20170920屏幕快照2017-09-20下午9.05.47.png)

JDBC连接桥接模式，相对之前，只是属性设置不一样。

### 桥接模式总结
桥接模式和适配器模式用于设计的不同阶段，桥接模式用于系统的初步设计，对于存在两个独立变化维度的类可以将其分为抽象化和实现化两个角色，使它们可以分别进行变化；而在初步设计完成之后，当发现系统与已有类无法协同工作时，可以采用适配器模式。但有时候在设计初期也需要考虑适配器模式，特别是那些涉及到大量第三方应用接口的情况。

在软件开发中如果一个类或一个系统有多个变化维度时，都可以尝试使用桥接模式对其进行设计。桥接模式为多维度变化的系统提供了一套完整的解决方案，并且降低了系统的复杂度。

**使用场景：**
* 如果一个系统需要在抽象化和具体化之间增加更多的灵活性，避免在两个层次之间建立静态的继承关系，通过桥接模式可以使它们在抽象层建立一个关联关系。
* 拆分后两部分互不影响。
* 一个类有多个变化维度，且多个纬度都需要进行独立扩展。
* 不希望使用多继承，导致类增多情况。

**优点 ：**
* 分离抽象接口及其实现部分，使各自都有自己子类，方便组合。
* 在有的情况下可以取代多继承，多层继承方案违背了“单一职责原则”，复用性较差，且类的个数非常多，桥接模式是比多层继承方案更好的解决方法，它极大减少了子类的个数。
* 桥接模式提高了系统的可扩展性，在两个变化维度中任意扩展一个维度，都不需要修改原有系统，符合“开闭原则”。

**缺点：**
* 桥接模式的使用会增加系统的理解与设计难度，由于关联关系建立在抽象层，要求开发者一开始就针对抽象层进行设计与编程。
* 不容易识别系统的独立变化纬度，需要一定经验积累。
