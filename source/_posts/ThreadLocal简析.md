---
title: ThreadLocal简析
date: 2019-10-22 18:29:45
tags: 
  - Java 
categories:
  - 技术
description: 对 ThreadLocal 的简要分析。
---

首先先来看一段代码
```java
public class ThreadId {
    private static final AtomicInteger nextId = new AtomicInteger(0);

    private static final ThreadLocal<Integer> threadId =
        new ThreadLocal<Integer>() {
            @Override protected Integer initialValue() {
                return nextId.getAndIncrement();
        }
    };

    public static int get() {
        return threadId.get();
    }

    public static void main(String[] args) {
		for(int i = 0; i  < 10; i++) {
			new Thread(new Runnable() {
				
				@Override
				public void run() {
					System.out.println(get());
				}
			}).start();
		}
	}
}
```
然后控制台输出如下：
```java
1
5
6
8
2
0
4
3
9
7
```
不难发现，不同的线程获取到的`threadId`是不一样的，那么为什么会出现这种情况呢，所以需要分析下`ThreadLocal`这个类，看看`threadId.get()`的逻辑
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```
第一步获取当前线程实例，然后调用`getMap(t)`获取到`ThreadLocalMap`实例
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
`getMap(Thread t)`方法直接返回了当前线程实例中的`threadLocals`成员变量，继续
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
其实`threadLocals`是`ThreadLocal`中的静态内部类`ThreadLocalMap`类型，那么此时返回为`null`，接着调用`return setInitialValue();`
```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
此时调用了重写的`initialValue()`方法进行`nextId`的自增操作，接着调用`createMap(t, value);`
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
此时创建了一个`ThreadLocalMap`实例并复制给当前线程的`threadLocals`变量，再来看其创建过程
```java
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
这里创建了一个初始长度为`INITIAL_CAPACITY`即长度为 16 的`Entry`类型数组，然后通过`firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);`获取到数组下标，创建`new Entry(firstKey, firstValue)`赋值到`table`的`i`位置，`size`记录的是数组中被赋值的个数，`setThreshold(INITIAL_CAPACITY);`是计算`threshold`的大小，即`INITIAL_CAPACITY * 2 / 3`，当`size`达到这个值时，`table`数组将会被扩容。接着我们来看看`Entry`
```java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```
`Entry`是`ThreadLocal`的静态内部类，利用虚引用保存`ThreadLocal`实例，另外还保存了`value`值。
`get`方法的流程大致就是如此了，首先会获取当前类中的`ThreadLocalMap`类型的`threadLocals`成员变量，如果为空则调用初始化方法，并且创建一个`ThreadLocalMap`实例赋值给当前线程，然后经过取余计算获取对应数组的位置创建`Entry`实例并赋值。
`ThreadLocal`在`Android`中也有应用，在`Handler`机制中，当我们调用`Looper.prepare();`创建`Looper`的时候用到了`ThreadLocal`存储。
```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
从代码中可以看出，一个线程最多只能创建一个`Looper`实例，不然会抛出异常，那么我们接下里就来分析下`ThreadLocal`的`set`方法
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
逻辑和`get`方法一样先获取当前线程的`threadLocals`，如果为空和上面的流程一样，这里就不再讲了，接下来看看`map.set(this, value);`
```java
private void set(ThreadLocal key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
大致就是先根据`key`即`ThreadLocal`实例的`threadLocalHashCode`获取到需要赋值数组的位置，然后判断是否有实例存在，有则覆盖`value`值，没有则创建一个`Entry`实例赋值到对应位置，最后判断需不需要`resize`...
`ThreadLocal`在我看来其实就是作用于线程范围的类，适用于不同的线程需要创建不同的数据副本情况，比如说`Android`中的`Looper`，暂时还未想到在`Andriod`实际开发中需要用到`ThreadLocal`的业务场景。