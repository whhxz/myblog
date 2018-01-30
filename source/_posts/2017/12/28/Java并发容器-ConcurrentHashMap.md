---
title: Java并发容器-ConcurrentHashMap
date: 2017-12-28 10:22:09
categories: ['多线程学习']
tags: ['并发容器', 'ConcurrentHashMap']
---

jdk1.7中ConcurrentHashMap与jdk1.8中实现不一致，1.7中采用的事分段锁，此处使用的是1.8，利用CAS+Synchronized来保证并发更新的安全，当然底层采用数组+链表+红黑树的存储结构。

### 前期概念
在ConcurrentHashMap中有几个重要内部类：
* Node：普通的节点，key-value形式，用于链表节点存储。
* TreeNode：红黑树节点，Node子类。
* TreeBin：红黑树，通过传入TreeNode构造一颗红黑树。
* ForwardingNode：Node子类，辅助节点，用于扩容操作。

在ConcurrentHashMap中，最开始是采用链表存储数据，当数据过长（TREEIFY_THRESHOLD默认长度8），就会转换为红黑树来处理，将原有Node节点包装成TreeNode放入TreeBin中，然后由TreeBin完成红黑树的转换。
<!-- more -->
### 源码分析
容器构造方法：
```java
//创建一个空对象，什么也不做，在后续put的时候，执行初始化
public ConcurrentHashMap() {
}

 /**
 * 创建一个指定大小空map，不需要动态调整大小
 */
public ConcurrentHashMap(int initialCapacity) {计算值
    //判断初始化容量
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    //如果初始化容量大于最大容量（(1 << 30)>>>1）使用MAXIMUM_CAPACITY，否则通过一套算法返回计算值
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    //sizeCtl 控制标示符
    /*1. 负数表示正在进行初始化或者扩容
    * 2. -1 表示正在初始化
    * 3. -N 表示有N-1个线程正在进行扩容
    * 4. 正数或者0表示hash表还没被初始化，这个数值表示初始化或下一次进行扩容的大小
    */
    this.sizeCtl = cap;
}

/**
 * 通过传递的Map创建一个新Map
 */
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    //默认值16
    this.sizeCtl = DEFAULT_CAPACITY;
    //设置新值
    putAll(m);
}

/**
 * 创建一个带有指定初始容量、加载因子（加载因子阈值，用来控制重新调整大小。在每 bin 中的平均元素数大于此阈值时，可能要重新调整大小。）的Map
 */
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

/**
 * 创建一个带有指定初始容量、加载因子、并发级别的空Map
 *
 * @param initialCapacity 初始化指定容器大小
 * @param loadFactor 建立初始化表大小的加载因子（表密度）
 * @param concurrencyLevel 并发等级，预估并发线程数量
 */
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    //校验入参
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    //如果并发等级大于初始化容器大小，容器大小采用并发等级
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    //计算下一次扩充大小
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```
ConcurrentHashMap.put源码：
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //校验参数
    if (key == null || value == null) throw new NullPointerException();
    //计算key hash
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //判断tab是否为null，初始化table
        if (tab == null || (n = tab.length) == 0)
            //初始化table
            tab = initTable();
        //判断i位((n-1)&hash)是否有插入值
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //没有插入值，通过Unsafe在指定位置插入值
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //判断是否正在扩容
        else if ((fh = f.hash) == MOVED)
            //帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //对节点加锁
            synchronized (f) {
                //重新判断节点
                if (tabAt(tab, i) == f) {
                    //fn = f.hash >=0 表示现在为链表，讲节点插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果key一样
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                //替换值
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //判断是否树节点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //树节点插入
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //判断链表长度是否超过最大值8
                if (binCount >= TREEIFY_THRESHOLD)
                    //把链表转换为树结构
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //map size + 1
    addCount(1L, binCount);
    return null;
}
//初始化table
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //判断sizeCtl
        if ((sc = sizeCtl) < 0)
            //sizeCtl小于0表示正在进行初始化或者扩容，暂停当前正在执行的线程对象，并执行其他线程
            Thread.yield(); // lost initialization race; just spin
        //修改sizeCtl为-1，初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //判断sc值，也就是之前构造函数传入初始化容器打小生成的sizeCtl值,默认为16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    //设置节点
                    table = tab = nt;
                    //下次扩容的大小
                    sc = n - (n >>> 2);//相当于0.75*n 设置一个扩容的阈值
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

/**
 * Adds to count, and if table is too small and not already
 * resizing, initiates transfer. If already resizing, helps
 * perform transfer if work is available.  Rechecks occupancy
 * after a transfer to see if another resize is already needed
 * because resizings are lagging additions.
 * 新增统计，如果table太小还准备调整大小，初始化扩容。如果已经调整大小，帮助扩容。在扩容完毕后重新检查容量，因为扩容相对落后
 * @param x 需要增加的值
 * @param check 如果小于0，不检查调整，如果小于等于1只需要检查是否存在竞争 if <0, don't check resize, if <= 1 only check if uncontended
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        //counterCells不为null或者CAS修改baseCount值失败（并发的时候可能会出现失败）
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //as为null 或者 as长度小于1 或者修改CounterCell失败，
            //查看LongAdder
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        //统计长度
        s = sumCount();
    }
    //扩容操作
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

```
简述put，如果map未初始化，先初始化map。获取key的hash，依据hash与table.size - 1计算相对table的偏移量（(n - 1) & hash)），通过unsafe获取该偏移处的值，如果为null，表示可以直接插入，如果不为空，判断当前是否正在扩容，如果是帮助先帮助扩容，如果不是，把获取的值加锁，判断当前是否链表，之后判断该值hash是否和table一样。如果一样替换，否则在链表末尾添加。如果不是链表依据树添加节点，之后判断如果是链表的话，是否超过最大链表长度8，超过转换为红黑树。最后map长度+1。

ConcurrentHashMap.get源码：
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //获取key hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        //获取key所在值
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //如果获取的值key与传入的一致，直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //如果是树节点
        else if (eh < 0)
            //在树节点中查找节点
            return (p = e.find(h, key)) != null ? p.val : null;
        //上述链表如果没找到值，遍历后续节点，因为可能会出现计算hash重复情况
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
简述get：先是通用获取key的hash，之后通过计算偏移量找到对应的值，如果获取的值和返回一样直接返回，如果未找到对应的值，判断当前是否树结构，通过在树中找到对应值，如果都未找到，可能是链表中hash相同情况，遍历后续节点比较值。

ConcurrentHashMap.remove源码
```java
public V remove(Object key) {
    return replaceNode(key, null, null);
}
final V replaceNode(Object key, V value, Object cv) {
    //获取key
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //直接获取hash对应的值，如果如果不存在直接返回
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        //判断当前是否在扩容
        else if ((fh = f.hash) == MOVED)
            //帮助扩容
            tab = helpTransfer(tab, f);
        else {
            //存在的值
            V oldVal = null;
            boolean validated = false;
            //锁住通过hash获取到的值
            synchronized (f) {
                //双重校验
                if (tabAt(tab, i) == f) {
                    //如果是链表
                    if (fh >= 0) {
                        validated = true;
                        //循环链表修改值
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    //判断是否树结构，树结构删除数据
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

ConcurrentHashMap.size源码：
```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        //遍历，所有counter求和
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}


```
在上述put代码中，在代码末尾有addCount操作，里面就有更新当前map长度。因为在使用size的时候，在此期间是可能存在put等其他操作，可能会出现结果不准。addCount中会判断当前是否需要扩容。

在JDK1.8 之后，如果需要返回size建议使用mappingCount()。

ConcurrentHashMap主要数据放入了table数组中，通过传入的key进行2次hash计算获取当前key应该存储在该数组所在位置，如果该位置为null，那么通过CAS设置值，如果该位置有值说明出现了hash碰撞，那么判断当前的key和已经存在的key是否一致，如果一致修改值，如果不一致放入链表末尾。当链表长度大于8时，会把该链表转换为红黑树。

参考：
http://cmsblogs.com/?p=2283
