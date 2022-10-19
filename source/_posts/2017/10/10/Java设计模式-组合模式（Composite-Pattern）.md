---
title: Java设计模式-组合模式（Composite Pattern）
date: 2017-10-10 20:27:17
categories: ['设计模式']
tags: ['设计模式', '组合模式', 'Composite Pattern']
---

### 树形结构的处理--组合模式
> 组合多个对象形成树形结构以表示具有“整体—部分”关系的层次结构。组合模式对单个对象（即叶子对象）和组合对象（即容器对象）的使用具有一致性，组合模式又可以称为“整体—部分”(Part-Whole)模式，它是一种对象结构型模式。

组合模式中，使得客户端看来单个对象和对象的组合是同等的。换句话说，某个类型的方法同时也接受自身类型作为参数。

组合模式UML如图所示：
![](/images/old/20171010屏幕快照2017-10-10下午9.54.17.png)

<!-- more -->
在组合模式中，通常有：
* 抽象构件（Component）：它可以是接口或抽象类，为叶子构件和容器构件对象声明接口，在该角色中可以包含所有子类共有行为的声明和实现。在抽象构件中定义了访问及管理它的子构件的方法，如增加子构件、删除子构件、获取子构件等。
* 叶子构件（Leaf）：它在组合结构中表示叶子节点对象，叶子节点没有子节点，它实现了在抽象构件中定义的行为。对于那些访问及管理子构件的方法，可以通过异常等方式进行处理。
* 容器构件（Composite）：它在组合结构中表示容器节点对象，容器节点包含子节点，其子节点可以是叶子节点，也可以是容器节点，它提供一个集合用于存储子节点，实现了在抽象构件中定义的行为，包括那些访问及管理子构件的方法，在其业务方法中可以递归调用其子节点的业务方法。

在组合模式中关键是抽象构件，既可以代表叶子也可以代表容器。还可以通过抽象构件进行统一操作。同时容器对象与抽象构件还是聚合关系，容器中既包含叶子也可以包含容器。

示例代码如下：
```Java
/**
 * 抽象构件
 */
abstract class AbstractComponent {
    /**
     * 构件操作
     */
    public abstract void operation();

    /**
     * 添加构件
     * @param component
     */
    public void add(AbstractComponent component){}

    /**
     * 删除构件
     * @param component
     */
    public void remove(AbstractComponent component){}
}

/**
 * 叶子构件
 */
class Leaf extends AbstractComponent {

    @Override
    public void operation() {
        System.out.println("叶子节点操作");
    }

    @Override
    public void add(AbstractComponent component) {
        throw new UnsupportedOperationException("不支持操作");
    }

    @Override
    public void remove(AbstractComponent component) {
        throw new UnsupportedOperationException("不支持操作");
    }
}

/**
 * 容器构件
 */
class Composite extends AbstractComponent {
    List<AbstractComponent> componentList = new ArrayList<>();

    @Override
    public void operation() {
        componentList.forEach(AbstractComponent::operation);
    }

    @Override
    public void add(AbstractComponent component) {
        componentList.add(component);
    }

    @Override
    public void remove(AbstractComponent component) {
        componentList.remove(component);
    }

    public AbstractComponent getChild(int i){
        return componentList.get(i);
    }
}
```
在示例代码中叶子构件中调用add、remove方法会抛出异常，不支持相关方法，在容器构件中有相关方法的实现。
### 透明组合模式和安全组合模式
如上实例代码中使用的是透明组合模式，由抽象类统一默认实现相关方法，这样有个确定是在子也构件使用相关方法的时候，会报错。如果不希望出现错误信息，可以是用安全组合模式。即抽象类删除掉不相干的方法只留一个operation，子叶构件直接实现不用而外添加相关方法。容器构件自己新增所需要的方法。这样有个缺点是，在使用的时候，必须指定子类。
### JDK中组合模式使用
在画图组件java.awt包中如：
* Component：抽象构件
* Container：容器
* Checkbox、Label、Button等叶子构件

Component如图：
![](/images/old/20171010屏幕快照2017-10-10下午9.31.21.png)

Container如图：
![](/images/old/20171010屏幕快照2017-10-10下午9.32.06.png)

该出使用的是安全组合模式。

### 组合模式总结
组合模式还可以用在该处
![](/images/old/20171010屏幕快照2017-10-10下午9.45.34.png)
公司组织结构队员的树形菜单。行政人员在下发通知的时候，可以直接下发到部门，也可以下发到子公司。

组合模式使用面向对象的思想来实现树形结构的构建与处理，描述了如何将容器对象和叶子对象进行递归组合，实现简单，灵活性好。由于在软件开发中存在大量的树形结构，因此组合模式是一种使用频率较高的结构型设计模式。

**使用场景：**
* 在具有整体和部分的层次结构中，希望通过一种方式忽略整体与部分的差异
* 在一个使用面向对象语言开发的系统中需要处理一个树形结构。
* 在一个系统中能够分离出叶子对象和容器对象，而且它们的类型不固定，需要增加一些新的类型。

**优点：**
* 组合模式可以清楚地定义分层次的复杂对象，表示对象的全部或部分层次。
* 可以一致地使用一个组合结构或其中单个对象，不必关心处理的是单个对象还是整个组合结构，简化了部分代码。
* 在组合模式中增加新的容器构件和叶子构件都很方便，无须对现有类库进行任何修改，符合“开闭原则”。
* 组合模式为树形结构的面向对象实现提供了一种灵活的解决方案，通过叶子对象和容器对象的递归组合，可以形成复杂的树形结构，但对树形结构的控制却非常简单。

**缺点：**
* 使用组合模式后，控制树枝构件的类型不太容易。比如需要某个容器组件全部为某个指定类型，需要在外部控制。
