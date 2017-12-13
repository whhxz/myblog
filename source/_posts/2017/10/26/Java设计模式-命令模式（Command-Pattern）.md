---
title: Java设计模式-命令模式（Command Pattern）
date: 2017-10-26 20:07:13
categories: ['设计模式']
tags: ['设计模式', '命令模式', 'Command Pattern']
---

### 命令模式
>将一个请求封装为一个对象，从而让我们可用不同的请求对客户进行参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。命令模式是一种对象行为型模式，其别名为动作(Action)模式或事务(Transaction)模式。

UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27上午9.28.37.png)
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27上午9.44.24.png)
<!-- more -->
```Java
/**
 * 抽象命令
 */
abstract class Command {
    //命令执行方法
    abstract void execute();
}

/**
 * 调用者
 * 用于发起命令
 */
class Invoker {
    private Command command;

    public Invoker(Command command) {
        this.command = command;
    }

    /**
     * 业务方法，用于调用命令
     */
    public void call() {
        command.execute();
    }
}

/**
 * 具体命令类
 */
class ConcreteCommand extends Command {
    //接收者
    private Receiver receiver;

    public ConcreteCommand(Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    void execute() {
        receiver.action();
    }
}

/**
 * 接收者
   常常会将Command实现类直接实现具体的逻辑和这个角色有重合
 */
class Receiver {
    public void action() {
        System.out.println("执行");
    }
}


public class Main {
    public static void main(String[] args) {
        //接收者
        Receiver receiver = new Receiver();
        //命令
        Command command = new ConcreteCommand(receiver);
        //客户端直接执行具体命令方式
//        command.execute();

        //客户端通过调用者来执行命令
        Invoker invoker = new Invoker(command);
        invoker.call();
    }
}

```
* Command（抽象命令类）:Command（抽象命令类）：抽象命令类一般是一个抽象类或接口，在其中声明了用于执行请求的execute()等方法，通过这些方法可以调用请求接收者的相关操作。
* ConcreteCommand（具体命令类）：具体命令类是抽象命令类的子类，实现了在抽象命令类中声明的方法，它对应具体的接收者对象，将接收者对象的动作绑定其中。在实现execute()方法时，将调用接收者对象的相关操作(Action)。
* Invoker（调用者）：调用者即请求发送者，它通过命令对象来执行请求。一个调用者并不需要在设计时确定其接收者，因此它只与抽象命令类之间存在关联关系。在程序运行时可以将一个具体命令对象注入其中，再调用具体命令对象的execute()方法，从而实现间接调用请求接收者的相关操作。
* Receiver（接收者）：接收者执行与请求相关的操作，它具体实现对请求的业务处理。

命令模式的本质是对请求进行封装，一个请求对应于一个命令，将发出命令的责任和执行命令的责任分割开。每一个命令都是一个操作：请求的一方发出请求要求执行一个操作；接收的一方收到请求，并执行相应的操作。命令模式允许请求的一方和接收的一方独立开来，使得请求的一方不必知道接收请求的一方的接口，更不必知道请求如何被接收、操作是否被执行、何时被执行，以及是怎么被执行的。

命令模式的关键在于引入了抽象命令类，请求发送者针对抽象命令类编程，只有实现了抽象命令类的具体命令才与请求接收者相关联。在最简单的抽象命令类中只包含了一个抽象的execute()方法，每个具体命令类将一个Receiver类型的对象作为一个实例变量进行存储，从而具体指定一个请求的接收者，不同的具体命令类提供了execute()方法的不同实现，并调用不同接收者的请求处理方法。


#### 宏命令
宏命令(Macro Command)又称为组合命令，它是组合模式和命令模式联用的产物。宏命令是一个具体命令类，它拥有一个集合属性，在该集合中包含了对其他命令对象的引用。通常宏命令不直接与请求接收者交互，而是通过它的成员来调用接收者的方法。当调用宏命令的execute()方法时，将递归调用它所包含的每个成员命令的execute()方法，一个宏命令的成员可以是简单命令，还可以继续是宏命令。执行一个宏命令将触发多个具体命令的执行，从而实现对命令的批处理。
新增命令队列：
```java
/**
 * 命令队列，批量执行命令
 */
class MacroCommand {
    private List<Command>commandList = new ArrayList<>();

    public boolean addCommand(Command command){
        return commandList.add(command);
    }
    public boolean removeComand(Command command){
        return commandList.remove(command);
    }

    public void execute() {
        commandList.forEach(Command::execute);
    }
}
```
修改调用者Invoker：
```java
/**
 * 调用者
 * 用于发起命令
 */
class Invoker {
    private MacroCommand macroCommand;

    public Invoker(MacroCommand macroCommand) {
        this.macroCommand = macroCommand;
    }

    /**
     * 业务方法，用于调用命令
     */
    public void call() {
        macroCommand.execute();
    }
}
```
#### 实际举例
需要一个库存项目，对外提供库存新增、扣减等操作。因为库存的频繁查询，实际库存是放入缓存中，为了避免数据库频繁更新，影响接口效率，更新缓存后，记录更新日志，方便日后查询，已经数据恢复。设计如下代码，采用命令设计模式把数据库操作抽象为相应的命令，对于数据库的操作封装好数据后放入队列，由其他线程处理队列中数据库操作命令，在实际应用中，为了避免因数据库宕机导致队列丢失，可以把命令修改操作放入消息中间件。
UML类图：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27上午11.08.46.png)
```java
/**
 * 抽象命令
 */
abstract class Command {
    //命令共有方法
    public void updateCache(){
        //更新缓存，封装参数等，为后续操作做准备
        System.out.println("更新缓存");
    }

    //命令执行方法
    abstract void execute();
}

/**
 * 库存新增
 */
class IncreaseCommand extends Command {
    private DBLog dbLog;

    public IncreaseCommand(DBLog dbLog) {
        this.dbLog = dbLog;
    }

    @Override
    void execute() {
        System.out.println("缓存库存添加！！！");
        dbLog.insertLog();
        System.out.println("其他操作！！！");
    }
}

/**
 * 库存扣减
 */
class DecreaseCommand extends Command {
    private DBLog dbLog;

    public DecreaseCommand(DBLog dbLog) {
        this.dbLog = dbLog;
    }

    @Override
    void execute() {
        System.out.println("缓存库存扣减！！！");
        dbLog.insertLog();
        System.out.println("其他操作");
    }
}

/**
 * 库存调用
 */
class Invoker{
    private BlockingQueue<Command> queue = new LinkedBlockingQueue<>();

    public void put(Command command) throws InterruptedException {
        command.updateCache();
        queue.put(command);
    }

    public void call(){
        new Thread(() -> {
            while (true){
                try {
                    queue.take().execute();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

}

/**
 * 数据库操作
 */
class DBLog {
    private String name;
    private String arg;

    public DBLog(String name, String arg) {
        this.name = name;
        this.arg = arg;
    }

    public void insertLog() {
        System.out.println(name + "：数据库日志插入！！参数：" + arg);
    }
}

public class Main{
    public static void main(String[] args) throws InterruptedException {
        DBLog increase = new DBLog("增加库存", "1");
        DBLog decrease = new DBLog("扣减库存", "1");

        Command increaseCommand = new IncreaseCommand(increase);
        Command decreaseCommand = new DecreaseCommand(decrease);

        Invoker invoker = new Invoker();
        invoker.put(increaseCommand);
        invoker.put(increaseCommand);
        invoker.put(decreaseCommand);

        invoker.call();
        invoker.put(decreaseCommand);
    }
}
```

在实际应用中，在用JAVA写GUI时，一个按钮有相应的功能，如果需要做到用户自定义按钮功能，也可以使用命令模式。
### JDK中命令模式使用
在JDK中ThreadPoolExecutor可以视为命令模式，UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171027屏幕快照2017-10-27上午9.35.31.png)
在ThreadPoolExecutor中：
* ThreadPoolExecutor：调用者
* Runnable：抽象命令
* BlockingQueue：命令队列

在需要执行命令时，由客户端实现Runnable提交到ThreadPoolExecutor由JVM执行执行。

### 命令模式总结
命令模式是一种使用频率非常高的设计模式，它可以将请求发送者与接收者解耦，请求发送者通过命令对象来间接引用请求接收者，使得系统具有更好的灵活性和可扩展性。在基于GUI的软件开发，无论是在电脑桌面应用还是在移动应用中，命令模式都得到了广泛的应用。

**应用场景：**
* GUI 中每一个按钮都是一条命令。
* 系统需要将请求调用者和请求接收者解耦，使得调用者和接收者不直接交互。请求调用者无须知道接收者的存在，也无须知道接收者是谁，接收者也无须关心何时被调用。
* 系统需要在不同的时间指定请求、将请求排队和执行请求。一个命令对象和请求的初始调用者可以有不同的生命期，换言之，最初的请求发出者可能已经不在了，而命令对象本身仍然是活动的，可以通过该命令对象去调用请求接收者，而无须关心请求调用者的存在性，可以通过请求日志文件等机制来具体实现。
* 系统需要支持命令的撤销(Undo)操作和恢复(Redo)操作。
* 系统需要将一组操作组合在一起形成宏命令。

**优点：**
* 降低系统的耦合度。由于请求者与接收者之间不存在直接引用，因此请求者与接收者之间实现完全解耦，相同的请求者可以对应不同的接收者，同样，相同的接收者也可以供不同的请求者使用，两者之间具有良好的独立性。
* 新的命令可以很容易地加入到系统中。由于增加新的具体命令类不会影响到其他类，因此增加新的具体命令类很容易，无须修改原有系统源代码，甚至客户类代码，满足“开闭原则”的要求。
* 在需要的情况下，可以比较容易的将命令记入日志。
* 可以比较容易地设计一个命令队列或宏命令（组合命令）。
* 为请求的撤销(Undo)和恢复(Redo)操作提供了一种设计和实现方案（如之前库存通过日志恢复）。

**缺点：**
* 使用命令模式可能会导致某些系统有过多的具体命令类。因为针对每一个对请求接收者的调用操作都需要设计一个具体命令类，因此在某些系统中可能需要提供大量的具体命令类，这将影响命令模式的使用。
