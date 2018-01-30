---
title: Java线程池
date: 2018-01-15 11:08:58
categories: ['多线程学习']
tags: ['线程池']
---

一般使用Thread创建线程，如果频繁的创建销毁线程，系统资源比较浪费，线程启动调用的native方法，任务不能立即执行。
采用线程池可以通过重复利用已经创建的线程降低线程创建和销毁赵成的系统消耗，提高任务的响应速度，还可以对线程池中线程进行统一分配、调优和监控。

### ThreadPoolExecutor
手动通过创建ThreadPoolExecutor创建线程池，其构造方法参数有：
* corePoolSize：池中所保存的线程数，包括空闲线程。（不能小于0）
* maximumPoolSize：池中允许的最大线程数。（不能小于等于0，不能小于corePoolSize）
* keepAliveTime：当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。（不能小于0）
* unit：keepAliveTime 参数的时间单位
* workQueue：执行前用于保持任务的队列。此队列仅保持由 execute 方法提交的 Runnable 任务。（不能为null）
* threadFactory：执行程序创建新线程时使用的工厂。（不能为null）
* handler：由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。（不能为null）

threadFactory可以默认使用Executors.defaultThreadFactory()、handler可以默认使用ThreadPoolExecutor.defaultHandler（使用的是内部类ThreadPoolExecutor.AbortPolicy）。

ThreadPoolExecutor有几个关键常量：
```java
//默认为RUNNING(11100000000000000000000000000000)
//通过计算3高位表示线程运行状态，低29位位线程池中线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//32 - 3 = 29
private static final int COUNT_BITS = Integer.SIZE - 3;
//00011111111111111111111111111111（后续用于计算线程状态以及线程数量）
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
//运行状态存储在高阶
//11100000000000000000000000000000(32位)
private static final int RUNNING    = -1 << COUNT_BITS;//接收新的任务并且也会处理已经提交等待的任务
//00000000000000000000000000000000 关闭状态
private static final int SHUTDOWN   =  0 << COUNT_BITS;//不会接收新的任务，但会处理已经提交等待的任务
//00100000000000000000000000000000 停止状态
private static final int STOP       =  1 << COUNT_BITS;//不接受新任务，不处理已经提交等待的任务，而且还会中断处理中的任务
//01000000000000000000000000000000 整理状态
private static final int TIDYING    =  2 << COUNT_BITS;//所有的任务被终止，workCount为0，为此状态时将会调用terminated()方法
//01100000000000000000000000000000 结束状态
private static final int TERMINATED =  3 << COUNT_BITS;//terminated()调用完成
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

提交任务使用execute：
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     * 1. 如果运行的线程数corePoolSize，尝试开始一个新的线程，
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
     //获取线程池存储的值
    int c = ctl.get();
    //比较线程池中运行的线程数量与corePoolSize的值
    if (workerCountOf(c) < corePoolSize) {
        //如果少于corePoolSize，新增worker，成功直接返回，如果新增启动成功，直接返回
        if (addWorker(command, true))
            return;
        //如果新增失败获取最新的ctl值
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        //如果当前线程状态为运行状态，把当前任务放入队列中

        //获取最新状态
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            //如果当前线程池不是运行状态，删除任务成功
            //拒绝任务、删除队列中的任务
            reject(command);
        else if (workerCountOf(recheck) == 0)
            //如果线程池中工作线程数量为0
            //添加null，不占用corePoolSize，添加null的目的是为了启动一个新的worker线程用来可以消费队列中的任务
            addWorker(null, false);
    }
    //如果队列已经满了无法放入
    else if (!addWorker(command, false))
        //尝试新增worker通过maximumPoolSize验证是否超过上限，这个时候启动新线程，但是不能超过maximumPoolSize
        reject(command);
}
//core：true表示用corePoolSize作为范围，否则使用maximumPoolSize
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        //获取线程池存储的值
        int c = ctl.get();
        //通过位运行计算当前线程池运行状态默认为111
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 在线程正常运行时二进制值为111，此时首位为1表示负数，所以rs小于等于SHUTDOWN（0）表示线程池此时为非RUNNING状态
        // 如果状态不为SHUTDOWN或者firstTask不为空或者工作队列为空，直接返回false
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
        //新增线程工作数量For + CAS
        for (;;) {
            //获取当前线程工作数量
            int wc = workerCountOf(c);
            //如果数量大于CAPACITY存储上限，或者运行数量是否超过设置最大值
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                //操作设置最大值直接返回
                return false;
            //CAS尝试修改工作数量，成功直接跳出最外层循环（AtomicInteger.compareAndSet）
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //重新获取值
            c = ctl.get();  // Re-read ctl
            //CAS修改失败后，如果当前线程池状态已经发生了变化，跳出循环重试
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    //新增工作线程完毕后执行，（只是修改ctl的值）
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //Workder继承AQS，用于设置状态，通过线程池工厂创建线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                //获取当前状态
                int rs = runStateOf(ctl.get());
                //当前状态小于0（运行状态）、或者当前状态为0且传入Runnable为null
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    //重新检查线程是否在运行
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    //如果worker中数量大于largestPoolSize（默认0），设置largestPoolSize为worker中数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                //释放锁
                mainLock.unlock();
            }
            //如果添加成功，启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //如果线程启动失败
        if (! workerStarted)
            //处理添加失败的值，比如修改ctl中原本新增的值，从workers中删除之前添加的worker
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
execute其实就是修改ctl值，通过设置的值处理应该新启动线程还是放入队列中。

在创建Worker后，线程工厂创建线程启动调用Worker.run，之后调用的是runWorker方法。
```java
final void runWorker(Worker w) {
    //获取当前线程
    Thread wt = Thread.currentThread();
    //获取线程任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    //设置Worker状态为1
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //如果task为null直接从队列中获取task，这里就是之前addWorker(null, false)用处，启动一个线程从队列中获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            //中断处理
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //子类实现，当前类为空代码块
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //运行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //子类实现，当前类为空代码块
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //处理任务退出
        processWorkerExit(w, completedAbruptly);
    }
}
```
总结线程池新增任务：
在线程池中新增任务时：当前工作线程数量小于corePoolSize，启动新线程；如果当前线程数量大于corePoolSize，把线程放入任务队列中，放入成功后，验证当前工作线程数是否为0，如果为0启动一个新线程处理任务；如果线程放入队列中失败，表示任务放入速度大于任务完成速度，这个时候启动新线程，但是总数不能大于maximumPoolSize。

对于submit，其实就是把Callback封装成FutureTask，然后调用execute，返回封装的FutureTask

### 线程池队列

ArrayBlockingQueue：数组阻塞队列，需要指定大小
LinkedBlockingQueue：链表队列，默认为Integer.MAX_VALUE
PriorityBlockingQueue：优先级阻塞队列
SynchronousQueue：同步阻塞队列

### 线程池拒绝策略RejectedExecutionHandler

AbortPolicy（默认）：直接抛出异常
CallerRunsPolicy：通过调用者所在线程执行任务，其实就是传递过来的Runnable.run，直接调用run方法
DiscardOldestPolicy：它放弃最旧的未处理请求（下一个任务），然后重试 execute；如果执行程序已关闭，则会丢弃该任务。也就是直接获取线程池中下一个任务然后不处理，重新执行execute
DiscardPolicy：不处理，丢弃当前未处理任务

在使用工具类Executors，基本上也是直接使用线程池，需要注意使用的队列。

在使用线程池的时候需要注意如果在线程中放入了数据也就是使用了ThreadLocal，因为线程在执行完任务后并不会销毁，可能其他任务获取到了当前线程设置的值，建议在使用完毕后删除ThreadLocal中的数据。

参考：
* http://blog.csdn.net/xiaoxufox/article/details/52278508
* http://cmsblogs.com
