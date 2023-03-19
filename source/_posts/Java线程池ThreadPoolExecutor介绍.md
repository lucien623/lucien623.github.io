---
title: Java线程池ThreadPoolExecutor介绍
date: 2019-10-20 16:40:32
tags: 
  - Java 
categories:
  - 技术
description:  ThreadPoolExecutor 介绍。
---

线程池常用于需要频繁创建和销毁线程的情况，因为创建和销毁线程都需要消耗时间，如果经常性这样操作，对于性能肯定是非常有影响的，`ThreadPoolExecutor`能够帮助我们创建线程池，帮助我们管理维护线程，先看看`ThreadPoolExecutor`在`OkHttp`中的使用
```java
executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
```
分析下在创建`ThreadPoolExecutor`时的几个参数
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
`corePoolSize`：线程池中的核心线程数，除非设置了`allowCoreThreadTimeOut`，否则它们一旦创建就不会被销毁（the number of threads to keep in the pool, even if they are idle, unless {@code allowCoreThreadTimeOut} is set）

`maximumPoolSize`：线程池中的最大线程数量（the maximum number of threads to allow in the pool）

`keepAliveTime`：当线程池中的数量超过核心线程数时，闲置线程可以存活的最大时间（when the number of threads is greater than the core, this is the maximum time that excess idle threads will wait for new tasks before terminating.）

`unit`：`keepAliveTime`最大限制时间的单位（the time unit for the {@code keepAliveTime} argument）

`workQueue`：线程等待执行的队列，常见的有`ArrayBlockingQueue`(基于数组实现的 FIFO 阻塞队列)，`LinkedBlockingQueue`（基于链表实现的 FIFO 阻塞队列），`SynchronousQueue`（没有任何容量的阻塞队列），`PriorityQueue`（具有优先级的无限阻塞队列）（the queue to use for holding tasks before they are executed.  This queue will hold only the {@code Runnable} tasks submitted by the {@code execute} method.）

`threadFactory`：创建线程的工厂，当这个参数不传时，会用默认的`DefaultThreadFactory`作为线程创建工厂（the factory to use when the executor creates a new thread）

`handler`：默认是`new AbortPolicy();`，当任务队列已满并且无法创建新线程的时候会抛`RejectedExecutionException`异常（the handler to use when execution is blocked because the thread bounds and queue capacities are reached）

要提交任务只需调用`execute(Runnable runnable)`方法即可。关闭线程池有两种方法，`shutdown()`会中断没有正在执行任务的线程，正在执行的不中断,`shutdownNow()`会中断所有任务的线程。

线程池的执行流程大致是这样的：，当我们提交任务时，先去判断核心线程池是否已满，若未达到`corePoolSize`，那么会创建一个新线程执行任务，如果已经达到了核心线程数，那么便判断工作队列是否已满，如果没有满，那么将任务放入工作队列中，满了则继续下一个判断，即判断线程池的数量是否已经超过最大线程数`maximumPoolSize`，没有超过则创建一个新线程执行任务，如果超过的话就执行线程饱和策略。

除了自己创建线程池外，`Executors`也提供了四种快速创建线程池的方法：

*Executors.newFixedThreadPool();*
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
`newFixedThreadPool`创建了一个仅含有`nThreads`个核心线程的线程池，当核心线程池满时，新提交的任务会放入一个无大小限制的阻塞队列。

*Executors.newSingleThreadExecutor();*
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
`newSingleThreadExecutor`创建了只允许有一个线程的线程池，如果线程池已满，那么会把新提交的任务放入一个无大小限制的阻塞队列中，这些任务会顺序执行，不会产生并发问题。

*Executors.newCachedThreadPool();*
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
`newCachedThreadPool`创建的线程池没有大小限制，任何新提交的任务都会新创建线程执行，处于闲置状态的线程在 60s 后将会被销毁，`OkHttp`创建线程池的策略和这个相同，只不过`OkHttp`自己定义了一个创建线程的工厂。

*Executors.newScheduledThreadPool();*
```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```
`newScheduledThreadPool`创建的线程池核心线程数由自己定义，线程池大小没有任何限定，闲置线程只能存活 10 毫秒，几乎一闲置就会被回收，这种方式创建的线程池还可以执行定时任务。

创建线程池的策略大致如此，如果是 CPU 密集型任务，那么配置 CPU 核数 + 1 个线程数的线程池，如果是 IO 密集型，那么配置 CPU 核数 * 2 个线程数的线程池。