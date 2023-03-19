---
title: ConcurrentHashMap原理探析
date: 2019-10-17 17:27:13
tags: 
  - Java 
categories:
  - 技术
description: 对 ConcurrentHashMap 的原理探析。 
---

`HashMap`在我们的开发中可以说是很常见了，主要是用来存储键值对的数据结构，`HashMap`并不是线程安全的，如果要想实现线程安全，可以使用`Collections.synchronizeMap(hashMap) `的方式，当然`JDK`里也以提供了另外一个可以实现并发操作的键值对存储结构，那就是`ConcurrentHashMap`，那么接下来我们就来探析下他是如何实现线程安全的，看看`put`方法的实现机制。
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
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
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
首先获取`key`到`hashCode`，调用`spread`方法，将`hashCode`的高 16 位和低 16 位进行异或操作，再与`HASH_BITS`进行与操作，`HASH_BITS`的值是 0x7fffffff，也就是 32 位带符号的最大整数，效果等同于取余，异或主要是为了降低碰撞的几率。接着进入循环之中，首先判断`table`是否为空，如果为空的话，则进行初始化
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
这里其实也用到了`Unsafe`和 CAS 机制，如果不了解可以自行先去看下，因为我们是用无参构造方法创建的对象，那么`sizeCtl`变量的值就是 0，然后利用 CAS 操作将`sizeCtl`设置为 -1，此时若有其他线程进入此方法，则会执行`Thread.yield();`,使当前线程从运行状态变为就绪状态，再回到创建`table`的线程，创建了一个起始大小为`DEFAULT_CAPACITY = 16;`长度的`Node`数组，并设置`sizeCtl`的值为 12，到此`table`就创建完成了。再回到循环中，然后通过与数组长度与运算，根据内存地址判断相应位置上的值是否为空，如果为空的话则通过 CAS 机制对在相应地址上进行赋值
```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```
赋值成功则跳出当前循环，失败则继续执行循环，即当前的数组上的对应`Node`已经赋过值，接下来重点看这里`synchronized (f) `，`f`是当前数组中的节点，这里也可以看出`ConcurrentHashMap`是**以数组中的每一个元素作为分段锁的，分段锁的个数即为数组的长度**。`if (fh >= 0) `判断数组中节点的`hash`值是否大于 0，是的话表示当前的存储方式还是链表，那么会从链表中找是否有`key`相同的节点，如果有则替换，没有则加入链表尾部。如果值小于 0（确切的值是 -2）的话，表示当前已经用红黑树存储了，插入`TreeBin`中。接着判断`binCount`的值，如果大于`TREEIFY_THRESHOLD`的值即 8，那么执行`treeifyBin(tab, i);`
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```
先判断`tab`数组的长度，如果小于`MIN_TREEIFY_CAPACITY`即 64，那么不会进行红黑树的转换，而是会将数组扩容，否则会将数组`index`处的链表构建成一个`TreeBin`，即红黑树实例。