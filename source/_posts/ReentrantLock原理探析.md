---
title: ReentrantLock原理探析
date: 2019-10-16 15:26:08
tags: 
  - android
categories:
  - 技术
description: 对 ReentrantLock 原理的简要分析。
---
最近在对之前看过的一些源码进行回顾，今天主要是对 ReentrantLock 的源码分析。
`ReentrantLock`有两个构造方法，当我们通过无参构造方法创建时，内部会创建的是非公平锁实例，那么本文也以非公平锁的实现机制做分析。
```java
public ReentrantLock() {
    sync = new NonfairSync();
}
```
加锁的方法其实很简单，只需要调用`lock.lock();`就行了，接下来看看这个`lock()`方法里到底执行了啥
```java
public void lock() {
    sync.lock();
}
```
可以发现接着调用了`NonfairSync`实例中的`lock()`
```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
先来看`compareAndSetState(0, 1)`的逻辑
```java
protected final boolean compareAndSetState(int expect, int update) {
    return U.compareAndSwapInt(this, STATE, expect, update);
}
```
这个`U`是`Unsafe`的实例，那这个类到底是干嘛用的呢，简单来讲，这个类就是用来提供硬件级别的原子操作的，比如通过它你可以获取到某个属性的内存地址。这个地方还牵涉到 CAS(Compare And Swap) 机制，包含三个操作数，内存地址 V， 预期原值 A，新值 B，进行 CAS 操作的时候首先会将内存地址 V 中的值与预期原值 A 比较，如果相同，则将 V 中的值更新为 B，不同则执行相应逻辑，像这里如果 CAS 操作失败的话会返回 false。 `STATE`是什么呢？
```java
STATE = U.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
```
这个变量保存的就是`state`成员变量的内存地址，这些都定义在`NonfairSync`的父类`AbstractQueuedSynchronizer`中，`AbstractQueuedSynchronizer`是一个基于 FIFO 队列的同步框架，`state`表示的就是同步状态，如果值为 0 说明没有线程获取锁，反之则说明有线程持有锁。ok，当`compareAndSetState(0, 1)`返回`true`时，即表示`state`之前的值为 0，没有线程持有锁，当前线程获取锁成功，然后执行`setExclusiveOwnerThread(Thread.currentThread());`把当前线程设置到`exclusiveOwnerThread`变量中。我们再来看看返回`false`的情况，执行`acquire(1);`方法
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
首先会调用`tryAcquire(arg)`方法
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
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
这个方法是子类`NonfairSync`来实现的，首先获取`state`的值，如果是 0，表示之前持有锁的线程已经释放锁了，所以会尝试获取锁，反之不是 0 的话，会先判断当前线程是不是已经持有锁的线程，如果是的话 state 的值继续加 1，这里也说明`ReentrantLock`是可重入锁。那么我们假如当前线程不是持有锁的线程，所以这个方法最后会返回`false`，继续执行`addWaiter(Node.EXCLUSIVE)`
```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            U.putObject(node, Node.PREV, oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}
```
首先创建一个`Node`实例，会持有当前线程的实例
```java
Node(Node nextWaiter) {
    this.nextWaiter = nextWaiter;
    U.putObject(this, THREAD, Thread.currentThread());
}
```
然后是一个`for`循环，里面会判断尾节点是不是`null`，如果是`null`的话，则会初始化队列，并创建一个新的节点，头尾节点都指向它
```java
private final void initializeSyncQueue() {
    Node h;
    if (U.compareAndSwapObject(this, HEAD, null, (h = new Node())))
        tail = h;
}
```
继续执行循环中的代码，此时尾节点已经不为`null`，会把之前持有当前线程节点的前驱指向尾节点，并把这个节点设置成尾节点，原来的尾节点的后驱指向含有当前线程的节点。总结一下就是如果当前没有队列，则创建一个新队列，头节点为没有设置任何值的`Node`，尾节点为包含当前线程的`Node`；如果已经有队列的话，则会把创建的含有当前线程的节点放入队列的尾部。当创建的`Node`放入队列之后，我们再来看看`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`方法
```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```
循环中首先获取当前节点的前驱节点，判断前驱节点是否为头节点，如果是，执行`tryAcquire(arg)`方法，尝试获取锁，这里假设仍有其它线程持有锁，那么会执行第二个 if 判断，`shouldParkAfterFailedAcquire(p, node)`
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```
这个方法用来判断前驱`Node`的`waitStatus`，这里会把前驱节点的`waitStatus`设置成`Node.SIGNAL`，即等待获取锁的状态，设置成功会返回`true`，继续执行`parkAndCheckInterrupt()`
```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```
线程到这里就被阻塞了。
`lock.unlock();`方法的执行流程
```java
public void unlock() {
    sync.release(1);
}
```
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

来看看`Sync`中`tryRelease(arg)`的实现
```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
首先判断`c`的值是否为 0，如果不为 0，则表示返回`false`，执行结束，这里也说明了如果之前`lock()`方法被多次调用，那么`unlock();`也应该的调用相等次数才会尝试释放锁，ok，那么假设这里之前只调用了一次`lock()`方法，c 的值为 0，获取到`head`节点，如果头节点不为空，并且`waitStatus`不等于 0，执行`unparkSuccessor(h);`
```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        node.compareAndSetWaitStatus(ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node p = tail; p != node && p != null; p = p.prev)
            if (p.waitStatus <= 0)
                s = p;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
我们把头节点作为参数传入方法中，首先判断`waitStatus`的值，之前我们设置过的等待获取锁的状态的值为`-1`是小于 0 的，然后把头节点的`waitStatus`的值通过 CAS 值设置为 0，获取头节点的后驱节点，即刚才我们为获取到锁的子线程节点，最后刚才阻塞的节点被唤醒，我们会到刚才阻塞住的地方
```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```
唤醒之后会继续执行`for`循环里的逻辑，如果当前节点的前驱节点是节点，然后调用`tryAcquire(arg)`方法就可以获取到锁了，`setHead(node);`会把当前节点里面的前驱节点变量和当前持有线程的变量设置为`null`，当前节点会变成`head`节点。

非公平锁的逻辑大致就是如此了，`ReentrantLock`里不是还有个公平锁`FairSync`么，区别就在与非公平锁调用`lock()`就会执行`compareAndSetState(0, 1)`尝试获取锁，而公平锁`FairSync`则会判断当前是否有线程持有锁，没有的话还会判断当前队列是否存在等待获取锁的节点，有的话会把当前线程的`Node`放入队尾，一开始并不会去尝试获取锁。
