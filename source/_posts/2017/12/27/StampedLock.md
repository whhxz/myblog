---
title: StampedLock
date: 2017-12-27 14:58:47
categories: ['多线程学习']
tags: ['多线程', '同步', 'StampedLock']
---

在使用ReentrantReadWriteLock时，获取写锁时，不能存在任何其他锁。如果存在读超大与写，可能会出现获取写锁线程一直处在等待状态导致饥饿。
在JDK1.8中引入StampedLock，StampedLock控制锁有3种状态：写，读，乐观读。

所谓的乐观读模式，也就是若读的操作很多，写的操作很少的情况下，你可以乐观地认为，写入与读取同时发生几率很少，因此不悲观地使用完全的读取锁定，程序可以查看读取资料之后，是否遭到写入执行的变更，再采取后续的措施（重新读取变更信息，或者抛出异常） ，这一个小小改进，可大幅度提高程序的吞吐量！！
<!-- more -->
StampedLock官方例子
```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    void move(double deltaX, double deltaY) { // an exclusively locked method
        //获取写锁
        long stamp = sl.writeLock();
        //修改数据
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            //释放写锁
            sl.unlockWrite(stamp);
        }
    }

    double distanceFromOrigin() { // A read-only method
        //获取乐观读，返回一个标示用于后续判断是否发生了写操作
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        //验证乐观锁期间是否存在写锁
        if (!sl.validate(stamp)) {
            //如果存在写锁修改了数据，获取悲观读锁。此处也可以使用tryOptimisticRead然后通过CAS获取锁
            stamp = sl.readLock();
            //修改数据
            try {
                currentX = x;
                currentY = y;
            } finally {
                //释放锁
                sl.unlockRead(stamp);
            }
        }
        //计算值
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    void moveIfAtOrigin(double newX, double newY) { // upgrade
        // Could instead start with optimistic, not read mode
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    break;
                } else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}
```
暂时对StampedLock理解不够，后续理解后补充StampedLock使用。

可参考：
* http://blog.csdn.net/sunfeizhi/article/details/52135136
* http://www.importnew.com/14941.html
* https://blog.takipi.com/java-8-stampedlocks-vs-readwritelocks-and-synchronized/
