---
title: RxJava原理探析
date: 2019-10-15 11:43:55
tags: 
  - android
categories:
  - 技术
description: 对 RxJava 原理的简要分析。
---

先来看如何创建`Observable`实例
```java
Observable observable = Observable.create(new ObservableOnSubscribe() {
    @Override
    public void subscribe(ObservableEmitter e) throws Exception {
		e.onNext(1);
    }
});
```
通过调用`Observable`中的`create`静态方法
```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}

```
最终返回的是一个`ObservableCreate`实例，它关联了一个我们创建的`ObservableOnSubscribe`实例，`ObservableCreate`是`Observable`的子类。
接着创建`Observer`实例
```java
Observer observer = new Observer() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(Object o) {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onComplete() {

    }
};
```
接下来调用`observable.subscribe(observer);`，我们来看看`subscribe`方法中做了什么
```java
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
		
```
主要的就是`subscribeActual(observer);`这一步，它的实现在`ObservableCreate`类中
```java
 @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```
我们可以看到这个方法里首先创建了一个`CreateEmitter`实例，并把`observer`关联起来，然后调用`observer.onSubscribe(parent);`，即执行了我们开始创建`Observer`时候实现的`onSubscribe(Disposable d)`方法，并把`CreateEmitter`实例传递进去。这个`CreateEmitter`实现了`Disposable`接口。接下来执行`source.subscribe(parent);`即调用我们一开始创建`obervable`实例时 new 的`ObservableOnSubscribe`实例的`subscribe`方法，此时执行`e.onNext(1);`，即`CreateEmitter`中的`onNext`方法
```java
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```
可以看到最后还是调用了`observer.onNext(t);`。
接下来看看 RxJava 是如何进行线程切换的
```java
observable
   .subscribeOn(Schedulers.io())
   .observeOn(AndroidSchedulers.mainThread())
   .subscribe(observer);
```
这里添加了两行代码，`Schedulers.io()`返回是一个`IoScheduler`实例
```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
即把当前的`Observable`和`IoScheduler`实例关联到新创建的`ObservableSubscribeOn`实例中去，最后返回这个实例，这个类也是继承于`Observable`的。此时如果调用`.subscribe(observer)`，那么会走到这里
```java
@Override
public void subscribeActual(final Observer<? super T> s) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

    s.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
和之前类似，先是把`Observer`实例关联到新创建的`SubscribeOnObserver`实例中，然后调用`Observer`实例的`onSubscribe`方法，然后创建一个`SubscribeTask`实例，关联实例`parent`，这个`SubscribeTask`是一个实现了`Runnable`接口的类，`run()`中执行了`source.subscribe(parent);`这时就大概可以猜到它会放到一个子线程里去执行
```java
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```
接着看`scheduler.scheduleDirect(new SubscribeTask(parent))`
```java
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
我们来看`IoScheduler`中`createWorker()`是怎么实现的
```java
public Worker createWorker() {
    return new EventLoopWorker(pool.get());
}							
```
`pool.get()`是获取`CachedWorkerPool`实例，是个实例在`IoScheduler`实例初始化的时候被创建
```java
@Override
public void start() {
    CachedWorkerPool update = new CachedWorkerPool(KEEP_ALIVE_TIME, KEEP_ALIVE_UNIT, threadFactory);
    if (!pool.compareAndSet(NONE, update)) {
        update.shutdown();
    }
}
```
`CachedWorkerPool`顾名思义就是`ThreadWorker`的缓存池，用`ConcurrentLinkedQueue`保存，接着
```java
static final class EventLoopWorker extends Scheduler.Worker {
    private final CompositeDisposable tasks;
    private final CachedWorkerPool pool;
    private final ThreadWorker threadWorker;

    final AtomicBoolean once = new AtomicBoolean();

    EventLoopWorker(CachedWorkerPool pool) {
        this.pool = pool;
        this.tasks = new CompositeDisposable();
        this.threadWorker = pool.get();
    }

    @Override
    public void dispose() {
        if (once.compareAndSet(false, true)) {
            tasks.dispose();

            // releasing the pool should be the last action
            pool.release(threadWorker);
        }
    }

    @Override
    public boolean isDisposed() {
        return once.get();
    }

    @NonNull
    @Override
    public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
        if (tasks.isDisposed()) {
            // don't schedule, we are unsubscribed
            return EmptyDisposable.INSTANCE;
        }

        return threadWorker.scheduleActual(action, delayTime, unit, tasks);
    }
}
```
构造方法里调用了`pool.get()`
```java
ThreadWorker get() {
    if (allWorkers.isDisposed()) {
        return SHUTDOWN_THREAD_WORKER;
    }
    while (!expiringWorkerQueue.isEmpty()) {
        ThreadWorker threadWorker = expiringWorkerQueue.poll();
        if (threadWorker != null) {
            return threadWorker;
        }
    }

    // No cached worker found, so create a new one.
    ThreadWorker w = new ThreadWorker(threadFactory);
    allWorkers.add(w);
    return w;
}
```
这一步其实就是判断缓存队列是不是空的，如果不是，就从中取出一个，否则就创建一个并 add 到`allWorkers`中。
ok，`createWorker()`这一步终于走完了，接着创建`DisposeTask`实例并把之前的`Worker`和`Runnable`实例关联进去，下一步执行`w.schedule(task, delay, unit);`
```java
@NonNull
@Override
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    if (tasks.isDisposed()) {
        // don't schedule, we are unsubscribed
        return EmptyDisposable.INSTANCE;
    }

    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
```
用之前获取到的`ThreadWorker`实例调用`scheduleActual`
```java
@NonNull
public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
    Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);

    if (parent != null) {
        if (!parent.add(sr)) {
            return sr;
        }
    }

    Future<?> f;
    try {
        if (delayTime <= 0) {
            f = executor.submit((Callable<Object>)sr);
        } else {
            f = executor.schedule((Callable<Object>)sr, delayTime, unit);
        }
        sr.setFuture(f);
    } catch (RejectedExecutionException ex) {
        if (parent != null) {
            parent.remove(sr);
        }
        RxJavaPlugins.onError(ex);
    }

    return sr;
}
```
我们可以看到，最终提交到`executor`线程池中执行，最终才会执行`source.subscribe(parent);`
`subscribeOn(Schedulers.io())`分析完，接下来看`observeOn(AndroidSchedulers.mainThread())`是如何执行的
```java
private static final class MainHolder {

    static final Scheduler DEFAULT = new HandlerScheduler(new Handler(Looper.getMainLooper()));
}
```
```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}
```
```java
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
这里创建了一个`ObservableObserveOn`实例，继续来看`subscribeActual(Observer<? super T> observer)`
```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```
`scheduler.createWorker();`这里返回的是从`HandlerScheduler`里创建的`HandlerWorker(handler)`，这个`handler`是与主线程的`looper`绑定的。
```java
@Override
public Worker createWorker() {
    return new HandlerWorker(handler);
}
```
接着调用`source.subscribe`，`source`就是我们调用`.subscribeOn(Schedulers.io())`返回的`ObservableSubscribeOn`实例，回到上面讲过的调用流程了。接下来如果执行`e.onNext(1);`那么就是调用`ObserveOnObserver`实例中的`onNext`方法
```java
@Override
public void onNext(T t) {
    if (done) {
        return;
    }

    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
```
执行`schedule();`
```java
void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}
```
最终会执行如下
```java
@Override
ublic Disposable schedule(Runnable run, long delay, TimeUnit unit) {
   if (run == null) throw new NullPointerException("run == null");
   if (unit == null) throw new NullPointerException("unit == null");

   if (disposed) {
       return Disposables.disposed();
   }

   run = RxJavaPlugins.onSchedule(run);

   ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);

   Message message = Message.obtain(handler, scheduled);
   message.obj = this; // Used as token for batch disposal of this worker's runnables.

   handler.sendMessageDelayed(message, Math.max(0L, unit.toMillis(delay)));

   // Re-check disposed state for removing in case we were racing a call to dispose().
   if (disposed) {
       handler.removeCallbacks(scheduled);
       return Disposables.disposed();
   }

   return scheduled;
}
```
因为我们之前讲过`handler`是绑定主线程的，然后就会在主线程中执行`run()` 方法，线程切换就在这里完成，接着执行`ObserveOnObserver`中的`run()`方法
```java
@Override
public void run() {
    if (outputFused) {
        drainFused();
    } else {
        drainNormal();
    }
}
```
接下来会走`drainNormal();`
```java
void drainNormal() {
    int missed = 1;

    final SimpleQueue<T> q = queue;
    final Observer<? super T> a = actual;

    for (;;) {
        if (checkTerminated(done, q.isEmpty(), a)) {
            return;
        }

        for (;;) {
            boolean d = done;
            T v;

            try {
                v = q.poll();
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                s.dispose();
                q.clear();
                a.onError(ex);
                worker.dispose();
                return;
            }
            boolean empty = v == null;

            if (checkTerminated(d, empty, a)) {
                return;
            }

            if (empty) {
                break;
            }

            a.onNext(v);
        }

        missed = addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
```
这里就是从队列中取出消息，然后调用`onNext`方法。
总结：RxJava 的优点在于链式调用，逻辑非常清楚，切换线程也十分方便，也可以随时中断流程。