---
title: Java锁
date: 2017-12-18 09:16:49
categories: ['多线程学习']
tags: ['多线程', '同步', '锁']
---

在Java线程基础中使用过synchronized，在JDK5一起，同步都是基于synchronized，在场景非常复杂的地方，使用synchronized不方便，JDK5引入了Lock。在包java.util.concurrent.locks中就有锁相关的类。

### AbstractOwnableSynchronizer
抽象独占同步锁。提供设置当前拥有独占访问的线程，获取设置的独占线程。
只能子类构造。

### AbstractQueuedSynchronizer
抽象队列同步，继承AbstractOwnableSynchronizer。通过队列先进先出来实现等待队列的阻塞，内部维护一个线程链表Node。
AbstractQueuedSynchronizer内部通过Unsafe.compareAndSet（原子操作int）来操作内存，保证线程的同步。其他API可查看相关文档。

### AbstractQueuedLongSynchronizer
同AbstractQueuedSynchronizer，只不过AbstractQueuedLongSynchronizer内部通过long字段来实现原子操作。当创建需要 64 位状态的多级别锁和屏障等同步器时使用。
<!-- more -->
### Condition
实际是将Object中监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，通过和Lock.newCondition()结合使用。通过Condition实现生产者消费者模型。例子：
```java
public class Main {
    private ReentrantLock lock = new ReentrantLock();
    private Condition emptyCondition = lock.newCondition();
    private Condition fullCondition = lock.newCondition();
    private int count, putIndex, takeIndex;
    private String[] items = new String[100];

    public void addItem(String str) {
        try {
            lock.lock();
            //数组已经满了
            while (items.length == count) {
                System.out.println("数据已满");
                fullCondition.await();
            }
            //放置
            items[putIndex] = str;
            //如果已经到了末尾，重新放置
            if (++putIndex == items.length) putIndex = 0;
            count++;
            emptyCondition.signal();
            System.out.println("放入：" + str);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    public String takeItem() {
        String item = null;
        try {
            lock.lock();
            //数组为空
            while (count == 0) {
                System.out.println("数据为空");
                emptyCondition.await();
            }
            //取出
            item = items[takeIndex];
            if (++takeIndex == item.length())takeIndex = 0;
            --count;
            fullCondition.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        System.out.println("取出：" + item);
        return item;
    }


    public static void main(String[] args) throws Exception {
        Main main = new Main();
        for (int i = 0; i < 5; i++) {
            new Thread(() ->{
                while (true){
                    SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SS");
                    String format = simpleDateFormat.format(new Date());
                    main.addItem(format);
                    try {
                        TimeUnit.MILLISECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                while (true){
                    main.takeItem();
                    try {
                        TimeUnit.MILLISECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
}
```

### Lock
Lock 实现提供了比使用 synchronized 方法和语句可获得的更广泛的锁定操作。此实现允许更灵活的结构，可以具有差别很大的属性，可以支持多个相关的 Condition 对象。

### LockSupport
用来创建锁和其他同步类的基本线程阻塞原语。一般采用其中park、unparkt，作用和wart、notify类似，但其实现原理不一样，而且不需要依赖监视器，和wart、notify无交集使用更加灵活。
例子：
```java

public class Main {
    public static void main(String[] args) throws Exception {
        String blocker = "blocker";

        Thread thread = new Thread(() -> {
            System.out.println("线程开始");
            LockSupport.park(blocker);
            System.out.println("线程结束");
        });

        thread.start();
        TimeUnit.SECONDS.sleep(5);
        System.out.println("重新启动线程");
        //中断线程也会解除阻塞，不会报错
//        thread.interrupt();
        LockSupport.unpark(thread);
    }
}
```



### ReentrantLock
Lock子类，提供可重入互斥锁。如：
```java
public class Main {
    ReentrantLock reentrantLock = new ReentrantLock();

    public static void main(String[] args) throws Exception {
        new Main().lockTest();

    }

    public void lockTest() {
        reentrantLock.lock();
        try {
            System.out.println("do something");
        } finally {
            reentrantLock.unlock();
        }
    }
}
```
通过ReentrantLock.lock()获取锁，之后释放锁。释放锁最好放入finally，避免因为锁未释放导致出现问题。
在ReentrantLock内部抽象类Sync继承AbstractQueuedSynchronizer，用于同步控制，同时Sync有两个子类NonfairSync（非公平）、FairSync（公平）。

#### FairSync
使用公平锁，new ReentrantLock(true)，在使用公平锁获取锁时。

调用ReentrantLock.lock获取锁，因为是公平锁，最后调用的是FairSync.lock，FairSync调用父类AbstractQueuedSynchronizer.acquire获取锁，之后调用子类方法FairSync.tryAcquire 试图获取锁获取锁。
```java
protected final boolean tryAcquire(int acquires) {
    //获取当前线程
    final Thread current = Thread.currentThread();
    //获取当前线程锁状态
    int c = getState();
    //未锁定
    if (c == 0) {
        /*
         * 1. 判断链表中需要处理的是否当前线程（头尾一致都为null，或者下一个节点为当前线程）
         * 2. 尝试Unsafe修改当前状态为锁定状态，获取锁
         * 3. 设置当前线程为锁持有
         */
        if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //已经锁定，锁定线程为当前线程（重入）
    else if (current == getExclusiveOwnerThread()) {
        //重入计数
        int nextc = c + acquires;
        //数据溢出
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //设置重入值
        setState(nextc);
        return true;
    }
    //获取锁失败
    return false;
}
```
1. 获取锁成功后直接返回
2. 获取锁失败后，生成Node节点放入等待列表，AbstractQueuedSynchronizer.addWaiter

```java
private Node addWaiter(Node mode) {
    //生成节点 model为null
    Node node = new Node(Thread.currentThread(), mode);
    // 获取末节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //设置末节点为当前节点
        if (compareAndSetTail(pred, node)) {
            //设置上一个末节点下一个节点为当前节点
            pred.next = node;
            return node;
        }
    }
    //没有末节点，或者设置当前节点为末节点失败，自旋重试
    enq(node);
    return node;
}
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //如果链表未初始化完成
        if (t == null) { // Must initialize
            //初始化链表，设置链表头为一个空节点
            if (compareAndSetHead(new Node()))
                //设置头尾节点一致，进入第二次循环
                tail = head;
        } else {
            //设置当前节点上一个节点为末节点
            node.prev = t;
            //设置当前节点为末节点
            if (compareAndSetTail(t, node)) {
                //设置上一个末节点下一个节点为当前节点
                t.next = node;
                return t;
            }
        }
    }
}
```
设置链表完成后，开始循环处理队列，直到轮到当前节点AbstractQueuedSynchronizer.acquireQueued
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //获取当前节点上一个节点
            final Node p = node.predecessor();
            //判断上一个节点是否为头节点，如果是，尝试获取节点，逻辑在上述tryAcquire中
            if (p == head && tryAcquire(arg)) {
                //当前节点获取锁后，设置当前节点为链表头
                setHead(node);
                //把原头结点的下一个节点设置为null，方便GC回收
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
在获取锁失败后，调用shouldParkAfterFailedAcquire、parkAndCheckInterrupt设置阻塞当前线程。
```Java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取上一个节点线程等待状态，默认为0
    int ws = pred.waitStatus;
    //如果为SIGNAL（-1）直接返回，表示已经设置了节点等待状态
    if (ws == Node.SIGNAL)
        return true;
    //如果等待状态大于0，表示上个线程节点已经被取消CANCELLED、则跳过上一个节点，把当前节点向前移，直到出现不大于0的情况
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    //如果等于0一般为节点才初始化，设置状态为SIGNAL（-1）
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
//上一个节点状态为SIGNAL（-1），则设置阻塞当前线程。当
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    //判断当前线程是否已经中断，并复位线程中断状态，因为线程在中断状态时，park不生效，会立即执行，如果不复位，会导致外面一直循环获取锁，CPU占用过高
    return Thread.interrupted();
}
```
上述描述了线程节点加入链表，之后线程被阻塞，当线程解除阻塞后。重新获取锁，并返回线程中断状态。之后调用线程interrupt。
如果在上述，循环获取锁过程中，出现异常导致退出循环且未获取到锁，将会调用cancelAcquire
```Java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
    //设置当前节点线程为空
    node.thread = null;

    // 获取上一个节点
    Node pred = node.prev;
    //如果等待状态大于0，表示上个线程节点已经被取消CANCELLED、则跳过上一个节点，把当前节点向前移，直到出现不大于0的情况
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    //设置当前节点等待状态为取消-1
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    //如果当前节点是更节点，设置根节点为上一个节点
    if (node == tail && compareAndSetTail(node, pred)) {
        //设置原来上一个节点的下个节点为null
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        //如果上一个节点不为头，且线程不为null，且上一个节点等待状态为SIGNAL
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            //获取下一个节点
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                //设置之前的上个节点的下一个节点为当前节点的下一个节点，这里没有设置下个节点的上一个节点为pred
                //在后续解除阻塞时，是从后向前遍历，同时会判断当前节点等待状态是否小于0，这里当前节点状态已经设置为0，当前节点会被跳过
                compareAndSetNext(pred, predNext, next);
        } else {
            //如果为第二个节点，或者上一个节点等待状态不为SIGNAL，解除当前节点的下一个节点的阻塞状态
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
     //修改当前节点等待状态为0
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    //获取当前节点下一个节点
    Node s = node.next;
    //如果当前节点为null，或者下一个节点等待状态为0
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从后向前遍历线程链表，直到当前节点，这样是因为在enq插入节点时，为了保证能遍历所有的节点
        //如果从前遍历，会导致如果下一个节点为正在插入的新节点，会出现获取不到新节点的情况，极端情况下会出现新节点获取不到锁，一直被阻塞
        //也就是上一个节点可以保证存在而且是对的，但是下一个节点可能无法获取
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //如果存在下一个节点，解除阻塞
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

解锁过程，ReentrantLock.unlock，之后调用父类AbstractQueuedSynchronizer.release
```java
public final boolean release(int arg) {
    //尝试释放资源：判断是否当前线程释放锁，如果完全解锁即锁状态为0，设置当前线程为null
    if (tryRelease(arg)) {
        //获取等待链表头
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //解除下一个节点阻塞状态，如上
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
以上是公平锁加锁解锁过程，公平锁就是如果多个未获取锁的线程在等待锁，维护一段链表，把当前线程放入链表末尾，之后线程进入阻塞状态，直到线程被上一个节点解除阻塞。
#### NonfairSync
非公平锁，创建ReentrantLock默认就是非公平锁。
在非公平锁情况下，如果调用lock，最终调用的是NonfairSync.lock
```java
final void lock() {
    //尝试直接修改当前锁状态0->1，如果修改成功，直接设置拥有锁的线程为当前线程
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //修改失败调用父类AbstractQueuedSynchronizer.acquire()，父类会回调子类tryAcquire如下
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    //调用父类Sync.nonfairTryAcquire
    return nonfairTryAcquire(acquires);
}
//abstract static class Sync
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //尝试修改当前状态
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //是否重入锁
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
非公平锁在获取锁失败后，后续操作和公平锁一样，直接生产线程节点，放入队列末尾，阻塞等待解除。

公平锁和非公平锁区别在于：
公平锁：去获取锁时，多判断一次，当前线程节点是否是下一个获取锁的节点（就算锁已经释放，也不获取），按照链表先后顺序执行。
非公平锁：获取锁时，直接尝试获取锁，如果这时候锁已经释放，直接获取锁，如果获取失败，后续按照公平锁方式创建线程节点加入等待链表。

在ReentrantLock.tryLock()，实际上是调用Sync.nonfairTryAcquire，如上，直接尝试获取锁，如果失败，不进入等待队列。
调用ReentrantLock.tryLock(long timeout, TimeUnit unit)，获取锁时调用的tryAcquire，和tryLock调用Sync.nonfairTryAcquire直接尝试获取锁不一样，该方法受是否公平模式影响。获取失败后会进入链表，并设置阻塞时间。
```java
//实际调用的方法
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    //判断线程是否中断，还原中断状态抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    //尝试获取锁，失败后调用doAcquireNanos，在等待时间内获取锁
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    //计算等待结束时间
    final long deadline = System.nanoTime() + nanosTimeout;
    //生成线程等待节点
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        //CAS循环操作
        for (;;) {
            final Node p = node.predecessor();
            //如果当前节点为第二个节点，尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            //计算是否超时
            if (nanosTimeout <= 0L)
                return false;
            //同之前代码，设置等待状态，同时判断离结束时间是否小于spinForTimeoutThreshold（1000）
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                //设置阻塞线程时间
                LockSupport.parkNanos(this, nanosTimeout);
            //线程被唤醒后判断线程是否中断状态
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
