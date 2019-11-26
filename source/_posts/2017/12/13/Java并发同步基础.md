---
title: Java并发同步基础
date: 2017-12-13 09:09:23
categories: ['多线程学习']
tags: ['多线程', '同步', 'synchronized', 'volatile']
---

### volatile定义与实现原理
Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致的更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比锁更加方便。如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。(Java语言规范第三版volatile定义)

volatile主要作用为：
1. 禁止指令重排序优化
2. 保证内存一致性

因为在写操作volatile修饰的变量时，会多出一行lock汇编代码。
在cpu操作时，会把部分数据缓存到CPU缓存里（L1、L2等）并未实时写回内存，如果对声明了volatile的变量进行写操作，会将当前变量所在的缓存行数据写回到系统内存，在多处理器下，为了保证其他处理核心缓存一致，会实现缓存一致协议，保证其他处理器检查当前缓存是否过期，后续操作时，会重新从系统内存中读取最新数据到缓存。简单来说，修改了volatile修饰的变量时，把CPU缓存中修改后的值及时写入内存，同时其他处理器设置自己缓存过期，使用时重新从系统内存中读取。
<!-- more -->
编译器和处理器为了提高并行度，可能会对部分操作进行重排序，在单线程执行下，重排序并不会改变执行结果。编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作可能被编译器和处理器重排序。（as-if-serial）

参考：
* [聊聊并发（一）——深入分析Volatile的实现原理](http://www.infoq.com/cn/articles/ftf-java-volatile)
* [JVM执行篇：使用HSDIS插件分析JVM代码执行细节](http://www.infoq.com/cn/articles/zzm-java-hsdis-jvm)
* [深入理解Java内存模型（二）——重排序](http://www.infoq.com/cn/articles/java-memory-model-2/)

常规单例：
```java
class Singleton {
    private static Singleton singleton;

    private Singleton() {
    }

    public static Singleton getSingleton(){
        if (singleton == null){
            synchronized (Singleton.class){
                if (singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

    public void out(){
        System.out.println("xxx");
    }
}
```
在单例设计模式中，采用双重检查锁来创建单例对象。因为可能出现指令重排序而导致该代码中有个安全隐患。
在代码 **singleton = new Singleton();** 创建对象，其操作可拆解为：
1. 堆内存开辟空间准备初始化对象
2. 初始化对象
3. singleton引用指向这个堆内存空间地址

实际转换为汇编，不会仅仅这几个步骤。如果现在进行了指令重排序变为1、3、2，这并不影响最终执行结果，如果现在属性singleton引用指向了改内存地址，但是对象还未初始化完成，另外一个线程进入该方法后判断singleton不为空，返回的singleton对象调用其方法时就会出错，因为对象实际上并未初始化完成。解决办法，在singleton属性上加上volatile标识，（jdk1.5之后有效）或者使用临时变量，创建完成后在赋值到singleton。

内存不一致性举例：
```java
class TaskTest implements Runnable {

    private boolean stop;

    @Override
    public void run() {
        int i = 0;
        while (!stop) {
            i++;
        }
        System.out.println(i);
    }

    public void stop() {
        stop = true;
    }
}

public class Main {

    public static void main(String[] args) throws Exception {

        TaskTest taskTest = new TaskTest();
        new Thread(taskTest).start();
        TimeUnit.SECONDS.sleep(1);
        taskTest.stop();
    }
}
```
```java
//TaskTest编译后代码
class TaskTest implements Runnable {
    private boolean stop;

    TaskTest() {
    }

    public void run() {
        int i;
        for(i = 0; !this.stop; ++i) {
            ;
        }

        System.out.println(i);
    }

    public void stop() {
        this.stop = true;
    }
}
```
在上述例子中，主线程启动了一个子线程，子线程有个循环，通过判断属性stop是否退出循环，之后主线程修改属性stop值，在实际运行中，子线程并不会退出循环，这就可能是因为stop被缓存到CPU缓存中，导致内存中数据修改，但是CPU缓存中的数据为过期数据，使循环无法退出。解决办法在stop属性中加入volatile修饰。

### synchronized基本使用
synchronized是实现同步的基础，Java中每个对象都可以作为一个锁，当一个线程试图访问同步代码块时，必须先获取锁。具体使用有3种方法：
1. 对普通方法同步，锁为当前实例对象。
2. 对静态方法同步，锁为当前类的Class对象
3. 对代码块进行同步，锁为括号内的对象

```Java
class SynchronizedDemo{
    public void method1(){
        synchronized (this){

        }
    }

    public synchronized void method2(){

    }
    public static synchronized void method3(){
    }
}
```
对改代码编译后，对该代码进行反编译。
第一个方法反编译部分如下：
```java
public void method1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_1
         5: monitorexit
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return

```
代码中存在monitorenter（监视器锁进入），这个解释如下：
每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
* 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
* 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1。
* 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

monitorexit（监视器锁退出）：执行monitorexit的线程必须是objectref所对应的monitor的所有者。指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

所以Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。

第二个方法显示如下：
```java
public synchronized void method1();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED
  Code:
    stack=0, locals=1, args_size=1
       0: return
    LineNumberTable:
      line 6: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       1     0  this   Lcom/whh/SynchronizedDemo;
```
在反编译后的代码中有ACC_PUBLIC（该方法为PUBLIC）ACC_SYNCHRONIZED（该方法为同步方法），JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

第三个方法反编译与第二个类似只是多个ACC_STATIC（静态方法）描述。

### 锁的升级
synchronized内部依赖于monitor，而monitor依赖于操作系统Mutex Lock线程之间锁的切换成本非常高，所以synchronized在jdk1.6以前效率非常低，在jdk1.6以后，为了减少获取锁释放锁带来性能消耗，引入了偏向锁、轻量级锁。之后锁有了4种状态：无锁状态<偏向锁<轻量级锁<重量级锁。状态会随着竞争逐渐升级，但锁（偏向锁可能会变为无锁暂停）不会降级，目的是为了提高释放锁的效率。

HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：**对象头（Header）**、**实例数据（Instance Data）**和**对齐填充（Padding）**。HotSpot虚拟机的对象头(Object Header)包括两部分信息，第一部分用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，这部分数据的长度在32位和64位的虚拟机（暂 不考虑开启压缩指针的场景）中分别为32个和64个Bits，官方称它为**Mark Word**。
在32位JDK中存储如下：
![](http://image.whhxz.smallstool.cn/20171213屏幕快照2017-12-13下午2.04.12.png)

#### 偏向锁
在很多情况下，锁并没有线程竞争，经常由同一个线程获取，HotSpot作者为了让线程获取锁的代价更低而引入了偏向锁。可以通过JVM参数关闭偏向锁：XX:-UseBiasedLocking=false，程序会默认进入轻量级锁。

偏向锁获取流程图如下：
![](http://image.whhxz.smallstool.cn/20171213屏幕快照2017-12-13下午5.56.54.png)
判断是否为偏向锁，如果为偏向锁，判断对象头的Mark Word里线程是否指向当前线程。如果不是需要采用CAS竞争锁，获取失败表示有竞争，当竞争达到全局安全点（safepoint）时，升级锁为轻量级锁。

偏向锁释放：
偏向锁默认是不会释放，需要等到出现竞争时才会释放，当有其他线程尝试竞争偏向锁时，持有偏向锁的线程才会被释放。偏向锁释放，需要等待全局安全点（该点上没有正在执行的字节码）。先暂停拥有偏向锁的线程，检查线程是否活着，处于不活动状态，设置对象为无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行遍历偏向对象的锁记录，之后设置对象头的Mark Word要么查询偏向于其他线程，要么恢复到无锁或者升级为轻量级锁，最后唤醒暂停的线程（可以先看轻量级锁）。
* 这里释放应该是，通过出现竞争时，有判断逻辑处理当前锁是否需要重新变为无锁、重偏向锁、升级为轻量级锁。

锁转换：
![](http://image.whhxz.smallstool.cn/20171213屏幕快照2017-12-13下午3.52.56.png)

**偏向锁主要是为了优化同一线程频繁访问锁。**
#### 轻量级锁
轻量级锁加锁过程：
1. 在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。
2. 拷贝对象头中的Mark Word复制到锁记录中
3. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock Record里的owner指针指向object mark word。
4. 更新成功，者当前线程拥有该对象的锁。设置Mark Word锁标志为00.
5. CAS多次更新不成功，虚拟机检查对象Mark Word是否指向当前线程的栈帧，如果是表示当前线程已经获取了对象的锁，进入同步块执行。如果不是表示存在线程竞争，轻量级锁变为重量级锁，锁标志变为10。Mark Work中存储的是指向重量级锁的指针，后面等待锁的线程会进入阻塞状态。当前线程会使用CAS来获取锁，目的是为了不让线程阻塞。

轻量级锁解锁过程：
1. 通过CAS把线程中复制的Displaced Mark Word对象替换为当前的Mark Word
2. 替换成功，同步完成结束
3. 替换失败，说明有其他线程尝试获取锁，锁已经变为重量级锁，需要释放锁的同时唤醒其他被挂起的线程。

轻量级锁升级为重量级锁，是为了避免资源消耗。轻量级锁没有太多空间存储额外的状态，太多线程自旋太消耗CPU资源。可参考https://www.zhihu.com/question/41930877/answer/136699311

#### 重量级锁
重量级锁依赖monitor，当一个线程进入同步代码块，其他线程会被阻塞在外，当线程执行完毕后释放锁，同时唤醒其他等待线程。

锁之间图解：
![](http://image.whhxz.smallstool.cn/20171213v2-9db4211af1be81785f6cc51a58ae6054_r.jpg)


#### 锁比较
| 锁 | 优点| 缺点| 场景 |
| ------------- |:-------------| :-----| :----- |
| 偏向锁 | 资源消耗少，同步方法月非同步方法差距不大 | 如果存在线程竞争，锁撤销消耗资源 | 单一线程访问 |
| 轻量级锁 | 线程竞争响应时间快 | 如果一直得不到锁，CPU自旋消耗资源 | 同步代码块执行速度快，线程交替执行，竞争少 |
| 重量级锁 | 无自旋，CPU消耗少 | 线程阻塞，响应慢 | 追求吞吐量，代码块执行速度长 |



### 虚拟机其他优化
适应性自旋（Adaptive Spinning）：从轻量级锁获取的流程中我们知道，当线程在获取轻量级锁的过程中执行CAS操作失败时，是要通过自旋来获取重量级锁的。问题在于，自旋是需要消耗CPU的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费CPU资源。解决这个问题最简单的办法就是指定自旋的次数，例如让其循环10次，如果还没获取到锁就进入阻塞状态。但是JDK采用了更聪明的方式——适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

锁粗化（Lock Coarsening）：锁粗化的概念应该比较好理解，就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁。举个例子：
```Java
public class StringBufferTest {
    StringBuffer stringBuffer = new StringBuffer();

    public void append(){
        stringBuffer.append("a");
        stringBuffer.append("b");
        stringBuffer.append("c");
    }
}
```

锁消除（Lock Elimination）：锁消除即删除不必要的加锁操作。根据代码逃逸技术，如果判断到一段代码中，堆上的数据不会逃逸出当前线程，那么可以认为这段代码是线程安全的，不必要加锁。看下面这段程序：
```java
//禁用了偏向锁
public class SynchronizedTest02 {

    public static void main(String[] args) {
        SynchronizedTest02 test02 = new SynchronizedTest02();
        //启动预热
        for (int i = 0; i < 10000; i++) {
            i++;
        }
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            test02.append("abc", "def");
        }
        System.out.println("Time=" + (System.currentTimeMillis() - start));
    }

    public void append(String str1, String str2) {
        StringBuffer sb = new StringBuffer();
        sb.append(str1).append(str2);
    }
}
```

参考：
* http://www.cnblogs.com/paddix/p/5405678.html
* https://www.zhihu.com/question/57774162/answer/154298044
* https://www.zhihu.com/question/53826114/answer/236363126
