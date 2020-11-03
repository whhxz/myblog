---
title: Java基础-Queue（1）
date: 2020-11-02 14:29:56
categories: ['Java基础']
tags: ['Queue', '数据结构']
---
### 定义
一般情况下，队列为一种先进先出的线性表。队列只允许在尾端插入数据，前端删除数据。也存在双端队列，可以在头部和尾部进行插入删除数据。

<!-- more -->
### 队列实现
#### 链表实现
```java
/**
 * 自定义队列
 */
class Queue<T> {
    //队列头部
    private Node<T> head;
    //队列尾部
    private Node<T> tail;

    /**
     * 添加数据到队列
     *
     * @param val 数据
     * @return 是否添加成功
     */
    public boolean offer(T val) {
        /*
         * 判断尾节点是否为空
         * 空：表示队列无数据，设置首尾节点为当前节点
         * 非空：放入链表尾部
         */
        Node<T> node = new Node<>(val);
        if (tail == null) {
            head = node;
            tail = node;
        } else {
            tail.next = node;
            tail = node;
        }
        return true;
    }

    /**
     * 去除队列数据
     *
     * @return 队列数据，如果不存在返回null
     */
    public T poll() {
        Node<T> node;
        if (head == tail) {
            node = head;
            head = null;
            tail = null;
        } else {
            node = head;
            head = head.next;
        }
        return node == null ? null : node.val;
    }
}

/**
 * 队列的节点，一个链表
 */
class Node<T> {
    //下一个节点
    Node<T> next;
    //节点值
    T val;

    Node(T val) {
        this.val = val;
    }
}

public class Demo {
    public static void main(String[] args) {
        Queue<Integer> queue = new Queue<>();
        queue.offer(1);
        queue.offer(2);
        queue.offer(3);
        Integer num;
        while ((num = queue.poll()) != null){
            System.out.println(num);//输出顺序为1、2、3
        }
    }
}
```
#### 数组实现
```java
/**
 * 自定义队列 数组实现
 */
class Queue<T> {
    //固定队列大小
    private Object[] data;
    //头
    private int head;
    //尾
    private int tail;

    public Queue(int size) {
        if (size <= 1) {
            throw new IllegalArgumentException("队列长度不能小于等于1");
        }
        data = new Object[size];
        head = 0;
        tail = 0;
    }

    /**
     * 添加
     *
     * @param t 数据
     * @return 是否添加成功
     */
    public boolean offer(T t) {
        // 队列中无数据
        if (head == tail && data[head] == null){
            data[head] = t;
            return true;
        }
        int next = (tail + 1) % data.length;
        if (data[next] != null){
            return false;
        }
        data[next] = t;
        tail = next;
        return true;
    }

    /**
     * 获取队列数据
     *
     * @return 数据
     */
    public T poll() {
        Object res = data[head];
        data[head] = null;
        //队列中并非只有一个数据时
        if (head != tail){
            head = (head + 1) % data.length;
        }
        return res == null? null: (T)res;
    }
}


public class Demo {
    public static void main(String[] args) {
        Queue<Integer> queue = new Queue<>(5);
        queue.offer(1);
        queue.offer(2);
        queue.offer(3);
        Integer num;
        while ((num = queue.poll()) != null) {
            System.out.println(num);
        }
        queue.offer(4);
        queue.offer(5);
        queue.offer(6);
        queue.offer(7);
        queue.offer(8);
        queue.offer(9);
        while ((num = queue.poll()) != null) {
            System.out.println(num);
        }
    }
}
```

* 上述两种实现，都是作为参考实现。具体可能有边界值判断的问题存在

链表的实现队列长度是无限长，这和链表的特性有关。数组为指定大小的容器存储队列数据，也可以对数组进行扩容。

Queue主要有两个子接口Deque（双端队列）、BlockingQueue（阻塞队列）、AbstractQueue（抽象类）