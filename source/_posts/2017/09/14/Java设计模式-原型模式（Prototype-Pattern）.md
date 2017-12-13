---
title: Java设计模式-原型模式（Prototype Pattern）
date: 2017-09-14 09:17:55
categories: ['设计模式']
tags: ['设计模式', '原型模式', 'Prototype Pattern']
---

### 原型模式
> 原型模式要求对象实现一个可以“克隆”自身的接口，这样就可以通过复制一个实例对象本身来创建一个新的实例。这样一来，通过原型实例创建新的对象，就不再需要关心这个实例本身的类型，只要实现了克隆自身的方法，就可以通过这个方法来获取新的对象，而无须再去通过new来创建。

原型模式相对于工厂模式而言，本身就是一个工厂，不同的是，工厂模式是直接创建一个新实例，而原型模式是克隆本身实例生成一个相似而完全不同的新实例，在内存中拥有新的地址，且每个克隆对象都是相互独立的。
<!-- more -->
Java实例代码如下：
```Java
class ConcretePrototype implements Cloneable{
    private String name;
    private int age;
    private List<String> list;

    public ConcretePrototype(String name, Integer age, List<String> list) {
        this.name = name;
        this.age = age;
        this.list = list;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public List<String> getList() {
        return list;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    /**
     * 克隆
     * @return
     */
    public ConcretePrototype clone(){
        Object clone = null;
        try {
            //object中的clone
            clone = super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return (ConcretePrototype) clone;

    }
}

public class Main {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ConcretePrototype concretePrototype = new ConcretePrototype("whh", 1, new ArrayList<>());
        ConcretePrototype clone = concretePrototype.clone();
        System.out.println(clone == concretePrototype);//false
        System.out.println(clone.getName()==(clone.getName()));//true
        System.out.println(clone.getAge()==(clone.getAge()));//true
        System.out.println(clone.getList()==(clone.getList()));//true
    }
}
```
代码中使用的Object中的clone方法，对象必须实现Cloneable接口，接口Cloneable的作用在于，在运行期间通知Java虚拟机安全的在这个类上使用clone方法，通过调用clone方法得到当前对象的复制。如果没有实现Cloneable接口，在运行期间会抛出CloneNotSupportedException。

克隆需要满足的条件有：
1. 对任何对象都有x.clone() != x
2. x.clone().getClass() == x.getClass()
3. 如果对象的equals方法定义恰当，x.clone().equals(x)为true

### 浅克隆、深克隆
对于浅克隆：
> 被复制对象的所有变量都含有与原来的对象相同的值，而所有的对其他对象的引用仍然指向原来的对象。换言之，浅复制仅仅复制所考虑的对象，而不复制它所引用的对象。

上面的例子中，属性的判断都为true

对于深克隆：
> 被复制对象的所有变量都含有与原来的对象相同的值，除去那些引用其他对象的变量。那些引用其他对象的变量将指向被复制过的新对象，而不再是原有的那些被引用的对象。换言之，深复制把要复制的对象所引用的对象都复制了一遍。

解决深度克隆可以使用Java中序列化。如：
```java
class ConcretePrototype implements Cloneable, Serializable {
    private String name;
    private int age;
    private List<String> list;

    public ConcretePrototype(String name, Integer age, List<String> list) {
        this.name = name;
        this.age = age;
        this.list = list;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public List<String> getList() {
        return list;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    /**
     * 克隆
     *
     * @return
     */
    public ConcretePrototype clone() {
        Object clone = null;
        try {
            //object中的clone
            clone = super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return (ConcretePrototype) clone;

    }

    /**
     * 深克隆
     * @return
     */
    public ConcretePrototype deepClone() {
        ConcretePrototype concretePrototype = null;
        ObjectOutputStream outputStream = null;
        ObjectInputStream inputStream = null;
        try {
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            outputStream = new ObjectOutputStream(byteArrayOutputStream);
            outputStream.writeObject(this);

            inputStream = new ObjectInputStream(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));
            concretePrototype = (ConcretePrototype) inputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            //关闭流
            if (outputStream != null){
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (inputStream != null){
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return concretePrototype;

    }
}

public class Main {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ConcretePrototype concretePrototype = new ConcretePrototype("whh", 1, new ArrayList<>());
        ConcretePrototype clone = concretePrototype.clone();
        System.out.println(clone == concretePrototype);//false
        System.out.println(clone.getName() == (concretePrototype.getName()));//true
        System.out.println(clone.getAge() == (concretePrototype.getAge()));//true
        System.out.println(clone.getList() == (concretePrototype.getList()));//true
        //深度克隆比较
        ConcretePrototype deepClone = concretePrototype.deepClone();
        System.out.println(deepClone == concretePrototype);//false
        System.out.println(deepClone.getName() == (concretePrototype.getName()));//false
        System.out.println(deepClone.getAge() == (concretePrototype.getAge()));//true
        System.out.println(deepClone.getList() == (concretePrototype.getList()));//false

    }
}
```
在深度克隆中，克隆的对象与元对象属性比较为false（基本类型比较除外）。深度克隆后生成的对象相对原类型完全独立，对克隆对象的修改，不会影响到原类型。

### 原型管理器
在使用克隆过程中，如果要控制克隆的数量，或者避免重复克隆，可以引入原型管理器，本质就是把克隆对象放入Map中，控制克隆生产和读取即可。在此不做举例。

### 原型模式总结
**原型模式优点：**
在在对象比较复杂时，通过复制一个已有对象，达到快速创建新对象。原型相对工厂而已，创建的是一个和原对象值一样的对象。

**原型模式缺点：**
如果对象引用了不支持序列化的间接对象，处理相对比较麻烦，在使用深度克隆的时候，如果对象结构比较复杂，序列化和反序列化比较消耗系统资源。
