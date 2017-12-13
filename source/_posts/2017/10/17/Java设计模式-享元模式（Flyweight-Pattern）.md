---
title: Java设计模式-享元模式（Flyweight Pattern）
date: 2017-10-17 14:02:58
categories: ['设计模式']
tags: ['设计模式', '享元模式', 'Flyweight Pattern']
---

### 享元模式
> 定义：运用共享技术有效地支持大量细粒度对象的复用。系统只使用少量的对象，而这些对象都很相似，状态变化很小，可以实现对象的多次复用。由于享元模式要求能够共享的对象必须是细粒度对象，因此它又称为轻量级模式，它是一种对象结构型模式。

享元模式以共享的方式高效地支持大量细粒度对象的重用，享元对象能做到共享的关键是区分了内部状态(Intrinsic State)和外部状态(Extrinsic State)。
* 内部状态：内部状态是存储在享元对象内部并且不会随环境改变而改变的状态，内部状态可以共享。
* 外部状态是随环境改变而改变的、不可以共享的状态。享元对象的外部状态通常由客户端保存，并在享元对象被创建之后，需要使用的时候再传入到享元对象内部。一个外部状态与另一个外部状态之间是相互独立的。

正因为区分了内部状态和外部状态，我们可以将具有相同内部状态的对象存储在享元池中，享元池中的对象是可以实现共享的，需要的时候就将对象从享元池中取出，实现对象的复用。通过向取出的对象注入不同的外部状态，可以得到一系列相似的对象，而这些对象在内存中实际上只存储一份。
<!-- more -->
享元模式UML如图：
![](http://otxnth5wx.bkt.clouddn.com/20171017屏幕快照2017-10-17下午3.41.00.png)
* 抽象享元（AbstractFlyweight）:通常是一个接口或抽象类，在抽象享元类中声明了具体享元类公共的方法，这些方法可以向外界提供享元对象的内部数据（内部状态），同时也可以通过这些方法来设置外部数据（外部状态）。
* 具体享元（ConcreteFlyweight）：它实现了抽象享元类，其实例称为享元对象；在具体享元类中为内部状态提供了存储空间。
* 非共享具体享元类(UnsharedConcreteFlyweight)：并不是所有的抽象享元类的子类都需要被共享，不能被共享的子类可设计为非共享具体享元类；当需要一个非共享具体享元类的对象时可以直接通过实例化创建。
* 享元工厂（FlyweightFactory）：享元工厂类用于创建并管理享元对象，它针对抽象享元类编程，将各种类型的具体享元对象存储在一个享元池中，享元池一般设计为一个存储“键值对”的集合（也可以是其他类型的集合）。

举例，在游戏中，有恢复药剂，分为HP恢复药剂以及MP恢复药剂，在没捡到一份药剂后，如果每次都创建一个药剂对象，会比较浪费系统资源，这个时候可以采用享元模式来设计：
![](http://otxnth5wx.bkt.clouddn.com/20171017屏幕快照2017-10-17下午3.33.44.png)
```java

import java.util.HashMap;
import java.util.Map;

/**
 * 药水
 */
abstract class AbstractPotion {
    /**
     * 药品名称
     */
    protected String name;

    public AbstractPotion(String name) {
        this.name = name;
    }

    /**
     * 使用药水
     */
    abstract void operation();

    public String getName() {
        return name;
    }
}

/**
 * 生命药剂
 */
class HealingPotion extends AbstractPotion {

    public HealingPotion(String name) {
        super(name);
    }

    @Override
    void operation() {
        System.out.println("使用" + name + "：你的HP恢复了");
    }
}

/**
 * 魔法药剂
 */
class MagicPotion extends AbstractPotion {

    public MagicPotion(String name) {
        super(name);
    }

    @Override
    void operation() {
        System.out.println("使用" + name + "：你的MP恢复了");
    }
}

/**
 * 药水工厂
 */
class PotionFactory {
    private PotionFactory(){
    }
    /**
     * 工厂药品存放
     */
    private static Map<String, AbstractPotion> potionMap = new HashMap<>();

    private final static Object potionBuild = new Object();

    /**
     * 药水类型
     */
    public enum PotionTypeEnum {
        //HP药剂
        HEALING_POTION,
        //HP药剂
        MAGIC_POTION
    }

    /**
     * 创建药品
     * @param potionType
     * @param name
     * @return
     */
    public static AbstractPotion buildPotion(PotionTypeEnum potionType, String name) {
        AbstractPotion potion = potionMap.get(potionType.name() + "_" + name);
        if (potion == null) {
            synchronized (potionMap) {
                potion = potionMap.get(potionType.name() + "_" + name);
                if (potion == null) {
                    switch (potionType) {
                        case HEALING_POTION:
                            potion = new HealingPotion(name);
                            potionMap.put(potionType.name() + "_" + name, potion);
                            break;
                        case MAGIC_POTION:
                            potion = new MagicPotion(name);
                            potionMap.put(potionType.name() + "_" + name, potion);
                            break;
                    }
                }
            }
        }
        return potion;
    }
}

public class Main {
    public static void main(String[] args) {
        AbstractPotion hp1 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "小型HP恢复药剂");
        AbstractPotion hp2 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "大型HP恢复药剂");
        AbstractPotion hp3 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "小型HP恢复药剂");
        AbstractPotion mp1 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "大型MP恢复药剂");
        AbstractPotion mp2 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "小型MP恢复药剂");
        AbstractPotion mp3 = PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "大型MP恢复药剂");

        System.out.println(System.identityHashCode(hp1));
        System.out.println(System.identityHashCode(hp2));
        System.out.println(System.identityHashCode(hp3));
        System.out.println(System.identityHashCode(mp1));
        System.out.println(System.identityHashCode(mp2));
        System.out.println(System.identityHashCode(mp3));
    }

}
```
在该例子中，提供工厂车间对象后，如果是同种药剂，返回的是同一个对象。非共享具体享元类，这里不做展示，和平常创建对象是一样的。

### 单纯享元模式、复合享元模式
* 在单纯享元模式中，所有的具体享元类都是可以共享的，不存在非共享具体享元类。（如上面例子）
* 将一些单纯享元对象使用组合模式加以组合，还可以形成复合享元对象，这样的复合享元对象本身不能共享，但是它们可以分解成单纯享元对象，而后者则可以共享。

复合享元模式例子：还是之前的系统，现在有个药剂师副业，可以生产出极度稀有的自定义药剂（这里只是举例理解复合享元模式，实际应用中应该不存在），系统能出产改药剂的人非常少。这时需要在系统中新增一个特殊药剂类：
```java
/**
 * 特殊药剂
 */
class SpecialPotion extends AbstractPotion{
    //药剂效果
    private List<AbstractPotion> effect = new ArrayList<>();

    public SpecialPotion(String name) {
        super(name);
    }

    public SpecialPotion(String name, List<AbstractPotion> effect) {
        super(name);
        this.effect = effect;
    }

    @Override
    void operation() {
        effect.forEach(AbstractPotion::operation);
    }
}
public class Main {
    public static void main(String[] args) {
        ArrayList<AbstractPotion> potionArrayList = new ArrayList<>();
        potionArrayList.add(PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "小型HP药剂"));
        potionArrayList.add(PotionFactory.buildPotion(PotionFactory.PotionTypeEnum.HEALING_POTION, "小型MP药剂"));
        SpecialPotion specialPotion = new SpecialPotion("特殊药剂", potionArrayList);
        specialPotion.operation();
    }
}
```
UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171017屏幕快照2017-10-17下午4.06.40.png)

新增的特殊药剂，有着同时恢复HP和MP功效，其内部属性effect集合中数据时共享的，由Factory创建。
### 在JDK中使用
在String和Integer等类中有使用享元模式。
先说String：
```java
public class Main {
    public static void main(String[] args) {
        String str1 = "abcd";
        String str2 = "abcd";
        String str3 = "ab" + "cd";
        String str4 = "ab";
        str4 += "cd";
        System.out.println(System.identityHashCode(str1));//1160460865
        System.out.println(System.identityHashCode(str2));//1160460865
        System.out.println(System.identityHashCode(str3));//1160460865
        System.out.println(System.identityHashCode(str4));//1247233941
    }
}
```
在Java中字符串被大量使用，为了避免每次都创建相同的字符串对象及内存分配，JVM内部对字符串对象的创建做了一定的优化，在内存中专门有一块区域用来存储字符串常量池。在第一次创建字符串"abcd"，会把该字符串对象放入常量池，第二次创建时，直接从常量池中获取改字符串。第三个在编译期间就会把改字符串合并为"abcd"所以是一样的，第四个，在实际运行中是通过StringBuilder来拼接的字符串之后toString返回。

* 注：在JDK1.7中, 已经把原本放在永久代的字符串常量池移出, 放在堆中。JDK 1.8 对 JVM 架构的改造将类元数据放到本地内存中。另外。将常量池和静态变量放到 Java 堆里。在这样的架构下。类元信息就突破了原来 -XX:MaxPermSize 的限制。如今能够使用很多其它的本地内存。这样就从一定程度上攻克了原来在执行时生成大量类的造成常常 Full GC 问题，如执行时使用反射、代理等。
* 注：除了字面常量值以外，常量池还可以容纳其它几种符号引用：类和接口的全限定名、字段名称和描述符、方法名称和描述符。

举例Integer：
```java
public class Main {
    public static void main(String[] args) {
        Integer int1 = 1;//自动装箱相当于Integer.valueOf(1)
        Integer int2 = 1;
        Integer int3 = 200;
        Integer int4 = 200;

        int i1 = 200;
        int i2 = 200;

        System.out.println(int1 == int2);//true
        System.out.println(int3 == int4);//false
        System.out.println(i1 == i2);//true
    }
}
```
在代码中int比较为true是因为int是基本类型，直接比较的值。两个Integer中比较一个为true，一个为false，查看源码得知，是因为Integer中使用了享元模式，在Integer中有个缓存池，保存了-128~127之间所有的Integer。

同样在项目中线程池、连接池等，都使用享元模式。

### 享元模式总结
享元模式是以节约内存、提高性能为出发点的设计模式。当系统中存在大量相同或者相似的对象时，享元模式是一种较好的解决方案，它通过共享技术实现相同或相似的细粒度对象的复用，从而节约了内存空间，提高了系统性能。相比其他结构型设计模式，享元模式的使用频率并不算太高。

享元模式和工厂模式区别在于，享元模式是内部使用了工厂模式来创建对象，是包含关系。
享元模式和单例模式区别在于，享元模式创建的对象在系统中并不是唯一存在，只是相似对象是唯一存在。而单例是类基本唯一存在，有且只有一个。相同点在于都能节约内存。

**使用场景：**
* 一个系统有大量相同或者相似的对象，造成内存的大量耗费。
* 对象的大部分状态都可以外部化，可以将这些外部状态传入对象中。（如药剂名字）
* 在使用享元模式时需要维护一个存储享元对象的享元池，而这需要耗费一定的系统资源，因此，应当在需要多次重复使用享元对象时才值得使用享元模式。

**优点：**
* 可以极大减少内存中对象的数量，使得相同或相似对象在内存中只保存一份，从而可以节约系统资源，提高系统性能。

**缺点：**
* 享元模式使得系统变得复杂，需要分离出内部状态和外部状态，这使得程序的逻辑复杂化。
* 为了使对象可以共享，享元模式需要将享元对象的部分状态外部化，而读取外部状态将使得运行时间变长。
