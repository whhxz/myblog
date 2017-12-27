---
title: Java读写锁
date: 2017-12-21 10:16:13
categories: ['多线程学习']
tags: ['多线程', '同步', '读写锁']
---

在多线程操作过程中，如果写少读多，采用ReentrantLock（排他锁），会比较浪费资源，这种情况下可以采用Java读写锁。
### ReadWriteLock
ReadWriteLock接口，定义了readLock、writeLock

### ReentrantReadWriteLock
重入读写锁，ReadWriteLock子类。对于ReadWriteLock，内部主要有：
* ReentrantReadWriteLock.ReadLock readerLock：读锁，Lock接口实现
* ReentrantReadWriteLock.WriteLock writerLock：写锁，Lock接口实现
* Sync sync：同步方法块，抽象接口AbstractQueuedSynchronizer子类，同时有NonfairSync、FairSync两个子类。

<!-- more -->
读写锁实例：
```java
public class Main {
    String[] items = new String[10];
    ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    Lock read = readWriteLock.readLock();
    Lock write = readWriteLock.writeLock();

    public String get(int index) {
        read.lock();
        try {
            return items[index];
        } finally {
            read.unlock();
        }
    }

    public void put(int index, String str) {
        write.lock();
        try {
            items[index] = str;
        } finally {
            write.unlock();
        }
    }

    public static void main(String[] args) throws Exception {
    }
}
```

### 读写状态设计
同步锁中同步器Sync是读写锁关键部分，其继承自AbstractQueuedSynchronizer（AQS）。同步器是通过一个状态表示锁被一个线程获取的次数，对于读写锁而言，需要在同步状态（state）维护读写两个状态，所以该状态被设计为按位切割，读写锁将改状态分为两个部分，高16位表示读，低16位表示写。
0000000000000000-00000000000000000，前面16位表示读，后面16位表示写。
在类初始化时定义了3个常量两个方法，如下：
```java
//状态转换
static final int SHARED_SHIFT   = 16;
//10000000000000000共享单元65536 作用于共享锁
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//最大重入次数65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
//1111111111111111排他标记65535 作用于排他锁
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

//共享锁状态统计，也就是共享锁线程获取数，如果当前状态为196608，二进制表示为110000000000000000，
//通过位运算向右移动16位，得到二进制11，表示有3个线程获取了读锁
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
//排他锁统计状态，通过与1111111111111111做与运算，得出当前线程是否有被写线程获取
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```
在写锁修改状态时直接通过当前状态state + acquires（通常为1），修改状态，也就是修改是低位状态，而对于读而言是state + SHARED_UNIT来修改状态，也就是修改的是高位状态。

### WriteLock
在写时，调用WriteLock.lock()，实际调用父类AbstractQueuedSynchronizer.acquire()，代码如下：
```java
//AbstractQueuedSynchronizer
public final void acquire(int arg) {
    //tryAcquire由子类Sync实现，调用的是AbstractQueuedSynchronizer.Sync.TryAcquire()
    //获取锁失败，生成等待线程队列，阻塞等待解除，解除后调用tryAcquire重新获取锁，这之后逻辑同ReentrantLock
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
//AbstractQueuedSynchronizer.Sync
//写锁获取锁时调用，尝试获取锁
protected final boolean tryAcquire(int acquires) {
    /*
    * 当存在其他线程读或者写、都会获取失败
    */
    //获取当前线程
    Thread current = Thread.currentThread();
    //获取当前状态
    int c = getState();
    //统计当前排他锁数量，实际调用c&1111111111111111
    int w = exclusiveCount(c);
    //如果有锁
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        //如果排他锁数量为0（此处表示为读锁），如果排他锁数量不为0，但是获取锁的线程不是当前线程，返回获取锁失败
        //判断是否有读锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //判断排他锁获取次数是否达到最大值65535
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        //设置当前状态
        setState(c + acquires);
        return true;
    }
    //判断写锁是否需要被阻塞，改方法为抽象方法，
    //由子类FairSync（调用父类AbstractQueuedSynchronizer.hasQueuedPredecessors判断当前线程是否等待节点第二个，
    //同ReentrantLock）、NonfairSync（对于读锁永远为false）实现
    //如果不需要被阻塞，设置锁状态
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    //修改状态成功后，修改当前获取锁的线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
```
流程图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171225屏幕快照2017-12-25下午6.34.52.png)
对于写锁而言，相对简单，只要当前存在锁，且不是当前线程获取，都会获取失败。如果存在锁，需要依据是否公平模式来判断是否有资格获取锁。

WriteLock.tryLock()：直接去获取锁，逻辑和tryAcquire类似，只是少了writerShouldBlock该步骤。
WriteLock.tryLock(long timeout, TimeUnit unit)：和ReentrantLock类似，先尝试获取锁，失败后进入等待线程链表，设置阻塞时间，到点唤醒线程获取锁。

WriteLock.unLock()：释放锁，调用的是父类AbstractQueuedSynchronizer.release代码如下：
```java
public final boolean release(int arg) {
    //尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //唤醒下一个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
//ReentrantReadWriteLock.Sync.tryRelease
protected final boolean tryRelease(int releases) {
    //判断获取锁的线程是否当前线程
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    //用于判断是否完全释放锁
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        //如果为0，设置当前修改锁状态为null
        setExclusiveOwnerThread(null);
    //修改状态
    setState(nextc);
    return free;
}
```
### ReadLock
读锁，调用ReadLock.lock后，调用的是父类方法AbstractQueuedSynchronizer.acquireShared()共享锁，代码如下：
```java
public final void acquireShared(int arg) {
    //尝试获取共享锁
    if (tryAcquireShared(arg) < 0)
        //获取共享锁
        doAcquireShared(arg);
}
//tryAcquireShared实际调用的是子类Sync.tryAcquireShared方法
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    //获取当前线程
    Thread current = Thread.currentThread();
    //获取当前锁状态
    int c = getState();
    //判断排它锁数量，如果有排他锁，但并不是当前线程获取锁，返回-1（失败）
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    //获取共享锁获数量，实际提供 状态c>>>16获取
    int r = sharedCount(c);
    //判断读是否需要阻塞，通过子类公平锁还是非公平锁处理
    if (!readerShouldBlock() &&
        //判断读锁最大获取线程是否超过最大值
        r < MAX_COUNT &&
        //尝试修改当前状态获取锁
        compareAndSetState(c, c + SHARED_UNIT)) {
        //如果当前线程锁获取成功
        //如果当前没有线程获取读锁
        if (r == 0) {
            //设置第一个读取线程为当前线程
            firstReader = current;
            //设置第一个读线程获取次数为1
            firstReaderHoldCount = 1;
        //如果第一个读线程为当前线程
        } else if (firstReader == current) {
            //第一个读线程获取次数+1
            firstReaderHoldCount++;
        } else {
            //用于统计读线程获锁次数，该值存储在线程中ThreadLocal
            HoldCounter rh = cachedHoldCounter;
            //如果cachedHoldCounter为空，或者缓存线程id不是当前线程id
            //也就是1 cachedHoldCounter未初始化，进入
            //2 cachedHoldCounter已经初始化，但是当前读锁线程id并不是缓存中的读锁线程id，进入
            if (rh == null || rh.tid != getThreadId(current))
                //设置缓存统计次数为当前线程中缓存次数，readHolds在Sync构造方法中初始化完成，ThreadLocal子类，用于记录线程读取次数
                //重新获取线程中的数据
                cachedHoldCounter = rh = readHolds.get();
            //如果为当前线程缓存的数据，且统计值为0，
            //因为ThreadLocalHoldCounter重写了ThreadLocal.initialValue方法，调用get方法时，会初始化HoldCounter
            else if (rh.count == 0)
                readHolds.set(rh);
            //统计+1
            rh.count++;
        }
        return 1;
    }
    //上述条件失败后调用
    return fullTryAcquireShared(current);
}
//尝试获取锁的完整版，用于CAS获取失败，或者tryAcquireShared获取失败是调用
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    //cas
    for (;;) {
        //获取锁当前状态
        int c = getState();
        //比较排他锁
        if (exclusiveCount(c) != 0) {
            //如果存在排它锁，且不是当前线程获取，返回-1获取失败
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        //判断读锁是否需要阻塞，依据是否公平锁处理，如果需要阻塞，执行下面步骤
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            //如果第一个读线程，不是当前线程
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        //判断是否超过最大读线程数量
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //尝试修改state获取锁
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            //获取锁成功
            //判断之前锁是否未被获取
            if (sharedCount(c) == 0) {
                //未被获取，设置第一个读取线程为当前线程
                firstReader = current;
                //设置第一个读线程获取次数为1
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                //如果当前线程为第一个获取读锁的线程，获取次数+1
                firstReaderHoldCount++;
            } else {
                //设置当前线程重入锁次数
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}

//在tryAcquireShared中如果获取锁失败，进入doAcquireShared
//AbstractQueuedSynchronizer.doAcquireShared
private void doAcquireShared(int arg) {
    //生产共享锁线程节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        //CAS
        for (;;) {
            //获取当前线程上一个节点
            final Node p = node.predecessor();
            //判断上一个节点是否头节点
            if (p == head) {
                //尝试获取共享锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    //获取成功
                    //设置头节点为当前线程传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //同ReentrantLock
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
//读线程获取锁后调用
private void setHeadAndPropagate(Node node, int propagate) {
    //置换头节点为下一个节点
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            //下一个节点为共享锁
            doReleaseShared();
    }
}
//释放共享锁
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //如果当前状态为SIGNAL状态，目的是为了唤醒当前后续的共享线程节点
            if (ws == Node.SIGNAL) {
                //设置当前节点状态为0，失败重试
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //如果设置节点状态成功，唤醒下一个节点
                unparkSuccessor(h);
            }
            //如果当前状态为0，设置为PROPAGATE
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //判断头节点是否被修改，如果被修改了，继续唤醒下一个节点
        //这里会出现唤醒的线程一直修改了头，就可能出现多个线程进入doReleaseShared方法，尝试修改下一个节点，这样唤醒节点更快吗？
        if (h == head)                   // loop if head changed
            break;
    }
}
```
尝试获取锁流程图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171225屏幕快照2017-12-25下午5.28.41.png)
获取锁失败后doAcquireShared如图：
![](http://otxnth5wx.bkt.clouddn.com/20171225屏幕快照2017-12-25下午6.30.33.png)

ReadLock.tryLock，调用的实际上是Sync.tryReadLock。比较简单，判断如果存在写锁切不是当前线程获取直接返回false，如果不存在写锁，直接修改锁状态，更新重入次数。
ReadLock.tryLock(long timeout, TimeUnit unit)，和之前类似，先尝试获取锁（受公平模式影响），如果失败放入链表阻塞，设置阻塞时间，等待唤醒。
ReadLock.unLock()，调用的父类AbstractQueuedSynchronizer.releaseShared，先tryReleaseShared尝试释放锁，后调用doReleaseShared()如上。tryReleaseShared调用比较简单，修改重入次数，修改锁状态。

### 公平锁、非公平锁
默认情况下创建时非公平锁。
公平锁：
读、写：都需要判断当前节点是否第二个节点、或者列表为null、或者就一个节点

非公平锁：
写：不需要阻塞
读：头节点不为空、下一个节点不为空、下一个不是共享节点、下一个节点线程不为空都成立下阻塞。其实就是如果下一个节点为非共享锁就需要阻塞。目的是为了不让写锁一直等待（此处下一个节点表示head下一个节点，即当前线程所在节点）。

* 注：非公平锁读是否阻塞时，在使用ReadLock.lock()会创建节点，节点的nextWaiter百分百是SHARE，那么在判断节点是否SHARE时肯定会返回True，WriteLock创建节点时，使用的是EXCLUSIVE，但是写锁由不会调用改方法，此处的作用是什么。很疑惑。可能是与其他方法结合使用吧。

### 读写锁交替使用
先使用读锁、后使用写锁，如下：
```java
ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
Lock read = readWriteLock.readLock();
Lock write = readWriteLock.writeLock();
read.lock();
write.lock();
```
上述代码write.lock会一直等待，因为读锁在获取锁时并不会修改属性exclusiveOwnerThread为当前线程，exclusiveOwnerThread为null，在write.lock时因为存在读锁，会判断当前获取锁的线程是否当前线程，返回-1获取锁失败，进入等待，这样就会导致线程死等。这种情况的出现，其他线程读锁无影响，写锁会无法获取。

先用写锁后用读锁，如下：
```java
ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
Lock read = readWriteLock.readLock();
Lock write = readWriteLock.writeLock();
write.lock();
read.lock();
```
上述代码并不会出现问题，在获取写锁后，读锁依然会正常获取，因为exclusiveOwnerThread在写获取锁时，会修改为当前线程，读锁在获取时会判断exclusiveOwnerThread是否为当前线程。此处先写锁后读锁，称为锁降级。在修改数据后需要读取数据，为了减少锁力度，可以采用该方法，读锁获取后释放写锁，当前线程以及其他线程都能拿到最新的数据。
