---
title: Java设计模式-装饰模式（Decorator Pattern）
date: 2017-10-12 16:36:30
categories: ['设计模式']
tags: ['设计模式', '装饰模式', 'Decorator Pattern']
---

### 扩展系统功能--装饰模式
> 定义：动态地给一个对象增加一些额外的职责，就增加对象功能来说，装饰模式比生成子类实现更为灵活。装饰模式是一种对象结构型模式。

装饰模式可以在不改变一个对象本身功能的基础上给对象增加额外的功能。装饰模式是一种替代继承的技术，它通过一种无须定义子类的方式来给对象动态增加职责，使用对象之间的关联关系取代类之间的继承关系。在装饰模式中引入了装饰类，在装饰类中既可以调用待装饰的原有类的方法，还可以增加新的方法，以扩充原有类的功能。

举例：
![](/images/old/20171012屏幕快照2017-10-12下午5.06.49.png)<!-- more -->
在这个里面孙悟空只是一种动物，在使用了七十二变之后，就可以变为其他事物。这里`七十二变`就是装饰器，装饰后就可以做别的事情了。这里只是举例，便于理解，实际设计需要依据情况来。
* 参考：[装饰设计模式](http://www.cnblogs.com/java-my-life/archive/2012/04/20/2455726.html)

在装饰模式中，实际UML图如下：
![](/images/old/20171012屏幕快照2017-10-12下午5.17.56.png)

* 抽象构件（Component）：它是具体构件和抽象装饰类的共同父类，声明了在具体构件中实现的业务方法，它的引入可以使客户端以一致的方式处理未被装饰的对象以及装饰之后的对象，实现客户端的透明操作。
* 具体构件（ConcreteComponent）：它是抽象构件类的子类，用于定义具体的构件对象，实现了在抽象构件中声明的方法，装饰器可以给它增加额外的职责（方法）。
* 抽象装饰器（Decorator）：它也是抽象构件类的子类，用于给具体构件增加职责，但是具体职责在其子类中实现。它维护一个指向抽象构件对象的引用，通过该引用可以调用装饰之前构件对象的方法，并通过其子类扩展该方法，以达到装饰的目的。
* 具体装饰器（ConcreteDecorator）：它是抽象装饰类的子类，负责向构件添加新的职责。每一个具体装饰类都定义了一些新的行为，它可以调用在抽象装饰类中定义的方法，并可以增加新的方法用以扩充对象的行为。

这里省略具体示例，直接分析JDK中应用。
### 在JDK中使用
在IO中InputStream相关类中是典型的装饰模式。如图：
![](/images/old/20171012屏幕快照2017-10-12下午6.05.34.png)
* InputStream：抽象构件，为子类提供统一接口
* FileInputStream、ByteArrayInputStream：抽象构件子类，实现父类相关接口
* FilterInputStream：装饰器，用于增强InputStream
* BufferedInputStream、PushbackInputStream：装饰器子类，增强InputStream。`BufferedInputStream `的作用是为另一个输入流添加一些功能，例如，提供“缓冲功能”以及支持“mark()标记”和“reset()重置方法”。带缓冲的流一次读取多个字节，相比InputStream一次读取一个字节，带缓存的磁盘操作减少了磁盘的操作次数，加快了读取速度。

```java
String fileName = "/Users/xuzhuo/Downloads/CentOS-7.0-1406-x86_64-DVD.iso";
byte[] bytes = new byte[1024];
//使用buffered读取
BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream(new File(fileName)));
long start = System.currentTimeMillis();
while (bufferedInputStream.read(bytes)!= -1){
}
long end = System.currentTimeMillis();
System.out.println(end - start);
bufferedInputStream.close();

//使用fileInputStream读取
bytes = new byte[1024];
FileInputStream fileInputStream = new FileInputStream(new File(fileName));
while (fileInputStream.read(bytes)!= -1){
}
System.out.println(System.currentTimeMillis() - end);
fileInputStream.close();
```
这里通过种读取文件测试，buffer在1700左右，直接读取在4000左右，使用装饰器对文件读取增强。

### 装饰模式总结
装饰模式和适配器模式都可以视为“包装”模式，它们都是通过封装其他对象达到设计的目的的。

理想的装饰模式在对被装饰对象进行功能增强的同时，要求具体构件角色、装饰角色的接口与抽象构件角色的接口完全一致。而适配器模式则不然，一般而言，适配器模式并不要求对源对象的功能进行增强，但是会改变源对象的接口，以便和目标接口相符合。

装饰模式分为两种：透明装饰模式、半透明装饰模式
* 透明的装饰模式也就是理想的装饰模式，要求具体构件角色、装饰角色的接口与抽象构件角色的接口完全一致
* 如果装饰角色的接口与抽象构件角色接口不一致，也就是说装饰角色的接口比抽象构件角色的接口宽的话，装饰角色实际上已经成了一个适配器角色，这种装饰模式也是可以接受的，称为“半透明”的装饰模式

透明装饰、半透明装饰、适配器可以理解为下图
![](/images/old/20171012屏幕快照2017-10-12下午6.34.13.png)
* 注：http://www.cnblogs.com/java-my-life/archive/2012/04/20/2455726.html

在InputStream中，FilterInputStream方法和InputStream中一样，可以视为透明装饰，但是在子类中新增了方法，就属于半透明模式。
在IO中Reader同样使用了装饰模式。

**使用场景：**
* 在不影响其他类的情况下，以动态、透明的方式给单个对象添加职责。
* 当不能采用继承的方式对系统进行扩展或者采用继承不利于系统扩展和维护时可以使用装饰模式。即系统中存在大量独立的扩展，为支持每一种扩展或者扩展之间的组合将产生大量的子类，使得子类数目呈爆炸性增长；或者类已定义为不能被继承（Final）。

**优点：**
* 对于扩展一个对象的功能，装饰模式比继承更加灵活性
* 可以通过一种动态的方式来扩展一个对象的功能，通过配置文件可以在运行时选择不同的具体装饰类，从而实现不同的行为。
* 可以对一个对象进行多次装饰，通过使用不同的具体装饰类以及这些装饰类的排列组合，可以创造出很多不同行为的组合，得到功能更为强大的对象。
* 具体构件类与具体装饰类可以独立变化，用户可以根据需要增加新的具体构件类和具体装饰类，原有类库代码无须改变，符合“开闭原则”。

**缺点：**
* 使用装饰模式进行系统设计时将产生很多小对象，大量的小对象在一定程度上会使用系统性能。
* 对于多次装饰的对象，容易出错，出错不好排查。
