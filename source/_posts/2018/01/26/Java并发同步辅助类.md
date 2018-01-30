---
title: Java并发同步辅助类
date: 2018-01-26 09:54:20
categories:
tags: ["并发同步", "工具类"]
---
### Semaphore
Semaphore是计数信号量。Semaphore经常用于限制获取某种资源的线程数量。也就是设置一个值，只允许知道数量的线程操作。
<!-- more -->
举例：
```java
public class SemaphoreDemo {
    private static Semaphore semaphore = new Semaphore(3);

    public static void main(String[] args) {
        IntStream.range(0, 5).forEach((i)->{
            new Thread(()->{
                while (true){
                    method();
                }
            }).start();
            try {
                TimeUnit.MILLISECONDS.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }

    public static void method(){
        try {
            //默认是1
            semaphore.acquire(1);
            System.out.printf("%s：申请获取资源成功\n", Thread.currentThread().getName());
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
            System.out.printf("%s：资源释放，剩余资源 %d\n", Thread.currentThread().getName(), semaphore.availablePermits());
        }
    }
}
```

### CyclicBarrier
同步屏障CyclicBarrier，CyclicBarrier表示一组线程在工作时，只有所有线程都达到某个点后才可以执行下一步，在最后一个线程未到达该点时，之前到达该点的线程都会被阻塞。
如：现在有个任务有3段，必须严格按照顺序执行，而没段任务内执行可以使用多线程加快执行速度。
```java
public class CyclicBarrierDemo {
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

    public static void main(String[] args) {
        Thread[] threads = new Thread[5];
        IntStream.range(0, 5).forEach((i) -> {
            threads[i] = new Thread(() -> {
                try {
                    method1();
                    System.out.printf("%s：执行完毕，进入等待状态，当前等待线程数：%d \n", Thread.currentThread().getName(), cyclicBarrier.getNumberWaiting());
                    cyclicBarrier.await();
                    method2();
                    System.out.printf("%s：执行完毕，进入等待状态，当前等待线程数：%d \n", Thread.currentThread().getName(), cyclicBarrier.getNumberWaiting());
                    cyclicBarrier.await();
                    method3();
                    System.out.printf("%s：执行完毕，进入等待状态，当前等待线程数：%d \n", Thread.currentThread().getName(), cyclicBarrier.getNumberWaiting());
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
            });
        });

        for (Thread thread : threads) {
            thread.start();
            try {
                TimeUnit.MILLISECONDS.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

    public static void method1() {
        System.out.printf("%s 执行步骤1\n", Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(3));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void method2() {
        System.out.printf("%s 执行步骤2\n", Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(3));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void method3() {
        System.out.printf("%s 执行步骤3\n", Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(new Random().nextInt(3));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```
CyclicBarrier在执行完毕后，可以重复使用，在使用CyclicBarrier时需要注意，执行的线数量需要大于等于设定的数量，不然会导致线程一直进入等待状态

### CountDownLatch
CountDownLatch，允许一个或多个线程一直等待，直到其他线程的操作执行完后再执行。
```java
public class CountDownLatchDemo {
    private static CountDownLatch countDownLatch = new CountDownLatch(3);

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            try {
                System.out.println("等待线程执行完毕...");
                countDownLatch.await();
                System.out.println("完成。");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        TimeUnit.SECONDS.sleep(1);
        IntStream.range(0, 3).forEach((i) -> new Thread(()->{
            try {
                System.out.printf("%s：线程开始执行\n", Thread.currentThread().getName());
                TimeUnit.SECONDS.sleep(1);
                System.out.printf("%s：线程执行完毕\n", Thread.currentThread().getName());
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start());
    }
}
```
在使用CountDownLatch时需要注意的是，CountDownLatch只能使用一次，使用之后CountDownLatch会失效。
CountDownLatch与CyclicBarrier区别在于，CyclicBarrier是一组线程互相等待直到都完成后，才继续后续步骤，而且CyclicBarrier是可以重用的。CountDownLatch是一个线程等待其他线程，直达到达指定值才开始执行，且不可重用。

### Exchanger
Exchanger允许两个线程到达共同设置的点时，交换数据。
```java
public class ExchangerDemo {

    static Exchanger<Integer> exchanger = new Exchanger<>();

    public static void main(String[] args) {
        new Thread(() -> {
            int num = 0;
            for (int i = 0; i < 5; i++) {
                try {

                    System.out.printf("%s：线程内部值为：%d \n", Thread.currentThread().getName(), num);
                    TimeUnit.SECONDS.sleep(1);
                    num = exchanger.exchange(num);
                    System.out.printf("%s：交换后线程内部值为：%d \n", Thread.currentThread().getName(), num);
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }).start();
        new Thread(() -> {
            int num = 1;
            for (int i = 0; i < 5; i++) {
                try {

                    System.out.printf("%s：线程内部值为：%d \n", Thread.currentThread().getName(), num);
                    TimeUnit.SECONDS.sleep(1);
                    num = exchanger.exchange(num);
                    System.out.printf("%s：交换后线程内部值为：%d \n", Thread.currentThread().getName(), num);
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }).start();
    }
}
```
