---
title: Java多线程学习记录（一）
date: 2017-12-11 14:42:56
categories: ['多线程学习']
tags: ['java', '多线程']
---

### 环境
 * jdk1.8

### 并发编程基本概念
为了利用多核处理器，采用多线程编程能节省大量运行时间，带来的缺点是会导致程序复杂度的提升。

**并行、并发**
并行：两个任务同时运行，如启动多个线程，每个线程运行各自的任务，在多核处理器中，不同的任务同时运行，可视为并行。（单核系统不存在并行）
并发：多个线程同时访问一个任务。

**同步、异步**
同步：调用一个方法，必须等待该方法返回后，才能继续下一步。
异步：调用方法后，必须等待方法的返回，继续执行下一步。在Java中被调用的方法，一般会在另一个线程里面继续执行。

**临界区**
用于多线程的互斥访问。如果有多个线程试图同时访问临界区，那么在有一个线程进入临界区后，其他试图访问的线程将被挂起，直到进入临界区的线程离开。临界区在被释放后，其他线程可以继续抢占，并以此达到对临界区的互斥访问。

**阻塞、非阻塞**
阻塞：在一个线程占用了临界区的资源，其他线程想要进入临界区需等进入临界区的线程释放资源，这时是其他等待的线程被阻塞挂起。
非阻塞：没有线程妨碍其他线程执行。

**死锁、饥饿、活锁**
死锁：单两个线程持有自己当前的锁，同时等待获取对方的锁，导致死锁。
饥饿：在非公平模式下，如果存在一个低优先级任务，同时存在大量高优先级任务，会导致任务低优先级任务长期获取不到资源，称之为饥饿。不过当高优先级任务完成后，低优先级任务还是有机会执行。
活锁：活锁、死锁本质上是一样的，原因是在获取临界区资源时，并发多个进程/线程声明资源占用(加锁)的顺序不一致，死锁是加不上就死等，活锁是加不上就放开已获得的资源重试（tryLock）。
死锁举例：
```java

import java.util.concurrent.TimeUnit;

class TaskA{
    public synchronized void method(TaskB b) {
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        b.method();
    }
    public synchronized void method(){
        System.out.println("任务A");
    }
}

class TaskB{
    public synchronized void method(){
        System.out.println("任务B");
    }
    public synchronized void method(TaskA a){
        a.method();
    }
}

public class Main {
    public static void main(String[] args) throws Exception{
        TaskA taskA = new TaskA();
        TaskB taskB = new TaskB();

        new Thread(() -> taskA.method(taskB)).start();
        new Thread(()-> taskB.method(taskA)).start();
    }
}
```
打开jvisualvm（JAVA_HOME/bin/jvisualvm）可以看到线程死锁了。
![](http://otxnth5wx.bkt.clouddn.com/20171211屏幕快照2017-12-11下午6.32.25.png)
饥饿举例：
```java
import javax.swing.*;
import java.awt.*;
import java.util.concurrent.TimeUnit;

class ProcessThread implements Runnable{
    JProgressBar progressBar;

    public JProgressBar getProgressBar() {
        return progressBar;
    }

    public ProcessThread(String name) {
        progressBar = new JProgressBar();
        progressBar.setString(name);
        progressBar.setStringPainted(true);
    }

    @Override
    public void run() {
        int c = 0;
        while (true){
            synchronized (Main.shareObj){
                if (c == 100){
                    c = 0;
                }
                c = c+ 10;
                progressBar.setValue(c);
                try {
                    //此处线程暂停，不释放锁，每次结束后由系统重新竞争锁，很大可能还是当前线程获取到锁。（偏向锁）
                    //在设计的时候，如果设计任务优先级执行，可能会重新高优先级任务一直运行
                    TimeUnit.MILLISECONDS.sleep(100);
                    //修改为下面的后，线程会暂停，释放锁，其他线程可以重新竞争锁，且当前线程不参与竞争。
//                    Main.shareObj.wait(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}


public class Main {
    public static final Object shareObj = new Object();

    public static void main(String[] args) throws Exception{
        JFrame starvation = new JFrame("Starvation");
        starvation.setDefaultCloseOperation(WindowConstants.EXIT_ON_CLOSE);
        starvation.setSize(300, 200);
        starvation.setLayout(new FlowLayout(FlowLayout.LEFT));

        for (int i = 0; i < 5; i++) {
            ProcessThread processThread = new ProcessThread("Thread-" + i);
            starvation.add(processThread.getProgressBar());
            Thread thread = new Thread(processThread);
            thread.start();
        }

        starvation.setLocationRelativeTo(null);
        starvation.setVisible(true);
    }
}
```
* 该例子可能并不恰当，不过饥饿很好理解。
* 通过图形化展示比较有趣，参考：http://www.logicbig.com/tutorials/core-java-tutorial/java-multi-threading/thread-starvation/

活锁举例：
```java
/**
 * 银行账户
 */
class BankAccount{
    private String name;
    private double balance;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getBalance() {
        return balance;
    }

    public void setBalance(double balance) {
        this.balance = balance;
    }

    public BankAccount(String name, double balance) {
        this.name = name;
        this.balance = balance;
    }

    private Lock lock = new ReentrantLock();

    /**
     * 取钱
     * @param amount
     * @return
     */
    boolean drawMoney(double amount){
        if (this.lock.tryLock()){
            try {
                TimeUnit.MILLISECONDS.sleep(100);
                balance -= amount;
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            } finally {
                lock.unlock();
            }
            return true;
        }
        return false;
    }

    /**
     * 存钱
     * @param amount
     * @return
     */
    boolean saveMoney(double amount){
        if (this.lock.tryLock()){
            try {
                TimeUnit.MILLISECONDS.sleep(100);
                balance += amount;
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            } finally {
                lock.unlock();
            }
            return true;
        }
        return false;
    }

    /**
     * 转账
     * @param destinationAccount
     * @param amount
     * @return
     */
    public boolean transfer(BankAccount destinationAccount, double amount){
        /*
         * 1. 先扣除当前账户
         * 2. 给目标账户加钱
         * 3. 目标账户加钱失败，回滚
         */
        if (this.drawMoney(amount)){
            if (destinationAccount.saveMoney(amount)){
                return true;
            } else {
                this.saveMoney(amount);
            }
        }
        return false;
    }
}

class TransferMoney implements Runnable{
    private BankAccount sourceAccount, destinationAccount;
    private double amount;

    public TransferMoney(BankAccount sourceAccount, BankAccount destinationAccount, double amount) {
        this.sourceAccount = sourceAccount;
        this.destinationAccount = destinationAccount;
        this.amount = amount;
    }

    @Override
    public void run() {
        while (!sourceAccount.transfer(destinationAccount, amount)){
            System.out.println("转账失败！！！重试");
        }
        System.out.printf("%s 余额 %.2f\t", sourceAccount.getName(), sourceAccount.getBalance());
        System.out.printf("%s 余额 %.2f\t", destinationAccount.getName(), destinationAccount.getBalance());
        System.out.println("转账成功！！！");
    }

}

public class Main {
    public static void main(String[] args) throws Exception{
        BankAccount accountMing = new BankAccount("小明", 10000.00);
        BankAccount accountRed = new BankAccount("小红", 20000.00);
        new Thread(new TransferMoney(accountMing, accountRed, 5000), "明->红").start();
        TimeUnit.MILLISECONDS.sleep(50);
        new Thread(new TransferMoney(accountRed, accountMing, 5000), "红->明").start();
    }
}
```
* 活锁不易察觉，活锁是有可能自己解开的，一般情况下存在活锁时，CPU会增高，可以通过日志记录来排查活锁。

**阿姆达尔定律、Gustafson**
阿姆达尔定律：http://ifeve.com/amdahls-law/
参考https://www.cnblogs.com/756623607-zhang/p/6850848.html

### Java线程
在Java中使用线程有两种方式
1. 实现Runnable接口
2. 继承Thread重写run方法
因为Java不支持多继承，多数情况下使用第一种方法。

#### Thread小析
Thread实际也实现了Runnable，同时拥有一个Runnable属性target，Thread实现的run方法实际是调用target的run方法。
```java
Thread.java

@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```
在new Thread的时候，会调用init初始化：
1. 设置线程名称
2. 设置线程组（如未设置，默认为安全管理器SecurityManager所在的线程组，如果SecurityManager线程组不存在，则设置为当前线程所在的线程组）
3. 权限校验（创建的线程对线程组的权限）
4. 设置新线程预计堆栈大小（默认0表示忽略堆栈大小）
5. 设置优先级、是否守护线程（默认同父线程）、目标类等相关操作

线程创建完成后，通过start启动线程。在启动线程前，可以通过setDaemon设置当前线程为守护线程。通过start启动后，最后会调用私有native方法start0，由虚拟机启动线程，调用run方法。在run方法执行完毕后，线程会自动关闭。
* 守护线程，当正在运行的线程都是守护线程时，Java 虚拟机退出。且守护线程创建的线程默认为守护线程，可在start前修改。

线程在创建到结束，总共有6个状态：
* NEW：线程创建完成，但还未启动。
* RUNNABLE：线程正在运行，在该状态时表示当前线程正在JVM中执行，但它可能正在等待操作系统中的其他资源，等待获取处理器调用
* BLOCKED：受阻塞并且正在等待监视器锁的某一线程的线程状态。处于受阻塞状态的某一线程正在等待监视器锁，以便进入一个同步的块/方法
* WAITING：不带超时线程等待状态，（不带超时的Object.wait、Thread.join、LockSupport.park都会导致线程进入该状态）
* TIMED_WAITING：带超时的等待状态（Thread.sleep、Object.wait、Thread.join、LockSupport.parkNanos、LockSupport.parkUntil都会导致线程进入该状态）
* TERMINATED：线程终止状态，表示线程已经执行完成。

Thread.interrupt()线程中断:
调用该方法后，并不是直接中断异常，如果线程内部未对中断进行处理，实际上中断无效。
```java
Thread thread = new Thread(() -> {
    int i = 1;
    for (int j = 1; j <= 10; j++) {
        i = i * j;
    }
    System.out.println(i);
});
thread.start();
thread.interrupt();
```
上诉例子中，线程永远都会计算完成后才结束。
```java
Thread thread = new Thread(() -> {
    int i = 1;
    while (true){
        if (Thread.interrupted()){
            break;
        }
        i++;
    }
    System.out.println(i);
});
thread.start();
TimeUnit.MILLISECONDS.sleep(1);
thread.interrupt();
```
在该例子中，线程内部有对中断状态进行处理，当线程发出中断信号后，会退出循环。Thread.interrupted方法会清除当前线程的中断状态。
需要注意的是，当线程为休眠（sleep或者wait）状态时，如果发出中断信号，会导致抛出InterruptedException异常，必须捕获处理。

Thread.join()等待该线程终止：
当线程之间需要协同操作时，当前线程需要等待其他完成后才能继续操作，可以采用join方法。
```java
Thread[] threads = new Thread[10];
threads[0] = Thread.currentThread();
for (int i = 1; i < threads.length; i++) {
    int j = i;
    threads[i] = new Thread(()->{
        try {
            threads[j - 1].join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName());
    }, "Thread-" + i);
}
for (int i = 1; i < threads.length; i++) {
    threads[i].start();
}
```
在该例子中，每个线程都是等上一个线程完成后(join)，在执行后续的操作。输出结果如下
```
Thread-1
Thread-2
Thread-3
Thread-4
Thread-5
Thread-6
Thread-7
Thread-8
Thread-9
```
如果不使用join，每个线程执行顺序是被打乱的。实际上join内部是通过wait来实现。

Thread.yield()暂停当前正在执行的线程对象，并执行其他线程：
调用该方法后，表示使当前线程让出CPU，让CPU重新分配执行线程，该线程会与其他线程继续竞争CPU资源。

### 线程返回Callable
之前实现的Runnable没有返回值。如果需要线程执行完毕后由返回值，可以使用Callable。
通常Callable和Futura一起使用，Callable计算处理返回结果，通过Futura获取Callable的返回值。举例：
```java
//创建callable，实现call方法，并返回
FutureTask<String> futureTask = new FutureTask<>(() -> "call");
new Thread(futureTask).start();
//等待线程执行完毕
TimeUnit.SECONDS.sleep(1);
//输出返回值
System.out.println(futureTask.get());
```
采用FutureTask的原因是因为，FutureTask实现了Runnable、Future两个接口。

#### 小析FutureTask
FutureTask有个Callable属性，由构造方法传入，或者由Executors生成Callable。outcome属性，存储Callable返回值
FutureTask拥有7个状态：
* NEW：新建
* COMPLETING：运行中
* NORMAL：正常
* EXCEPTIONAL：异常
* CANCELLED：取消
* INTERRUPTING：中断中
* INTERRUPTED：被中断

状态变化有4种情况：
* NEW -> COMPLETING -> NORMAL
* NEW -> COMPLETING -> EXCEPTIONAL
* NEW -> CANCELLED
* NEW -> INTERRUPTING -> INTERRUPTED

因为FutrueTask实现了Runnable接口，实际是通过JVM启动线程执行run方法。
FutrueTask的run方法实际上就是调用Callable的call方法，获取到方法返回值后，修改状态为COMPLETING，设置outcome为返回值，修改状态为NORMAL，通知其他等待的线程（在FutrueTask中有个链表维护等待的线程）。
FutrueTask通过get获取返回值，可以设置等待时间，之后会放入FutrueTask线程等待链表中。
FutrueTask其他方法如cancel之类的，可查看相关文档。
* 注：FutrueTask中状态的变化都是通过UNSAFE操作的，保证了线程的安全。

### 多线程异常
在使用Runnable的时候，如果线程内发生异常，并不会向主线程抛出异常，这样导致主线程无法感知子线程中异常。如果需要处理子线程异常，需要在run方法中try catch代码块。举例：
```java
//异常发生时catch并不会被执行
try {
    new Thread(()->{
        int i  = 1/0;
    }).start();
}catch (Exception e){
//            e.printStackTrace();
    System.out.println("发生异常");
}
```
```java
//改进需要在run方法中处理异常
new Thread(() -> {
    try {
        int i = 1 / 0;
    } catch (Exception e) {
        System.out.println("发生异常");
    }
}).start();
```
在FutrueTask.run方法中，调用Callable.call被try catch包裹
```java
//FutrueTask.run部分源码
try {
    result = c.call();
    ran = true;
} catch (Throwable ex) {
    result = null;
    ran = false;
    setException(ex);
}
```
catch后设置outcome为异常的值，同时设置当前状态为EXCEPTIONAL，当用户调用get时，判断当前状态，如果是异常状态，抛出该异常。

在开发过程中，经常需要统一处理相关异常。这可在Thread.setUncaughtExceptionHandler设置线程异常处理。如下：
```java
Thread thread = new Thread(() -> String.valueOf(1 / 0));
thread.setUncaughtExceptionHandler((t, e) -> System.out.println("发生了异常"));
thread.start();
```
