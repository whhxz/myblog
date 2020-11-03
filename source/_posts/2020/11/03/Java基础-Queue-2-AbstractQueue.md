---
title: Java基础-Queue（2）AbstractQueue
date: 2020-11-03 16:16:38
categories: ['Java基础']
tags: ['Queue', '数据结构', 'AbstractQueue']
---

在Java中Queue类主要存在两个子接口、一个抽象子类AbstractQueue。
AbstractQueue继承AbstractCollection和实现接口Queue，不允许存在null节点，存入和删除节点是如果为null会直接报错。

<!-- more -->
### AbstractQueue实现方法
* add，直接调用offer方法，如果成功返回true，失败抛出异常
* remove，直接调用poll方法，如果返回null抛出异常
* element，直接调用peek方法，如果返回为null，抛出异常，在Queue接口中该方法注释为，element和peek方法类似，都是只是查看队列头部数据，并不取出数据，区别在于peek方法如果没数据返回null，element没数据抛出异常。
* clear，循环调用poll方法，直到返回为null
* addAll，如果入参不能为null和本身，循环传入集合，调用add方法，只要有一个添加成功，返回true

### AbstractQueue子类
AbstractQueue子类有
> * **LinkedTransferQueue**、
* **SynchronousQueue**
* **PriorityQueue**
* **LinkedBlockingQueue**
* **PriorityBlockingQueue**
* **ArrayBlockingQueue**
* **LinkedBlockingDeque**
* **DelayQueue**
* **ConcurrentLinkedQueue**

其中部分之类实现了另外的Queue子接口。单一实现AbstractQueue子类有**PriorityQueue**、**ConcurrentLinkedQueue**。