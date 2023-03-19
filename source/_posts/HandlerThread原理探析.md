---
title: HandlerThread原理探析
date: 2019-10-12 19:01:50
tags: 
  - android
categories:
  - 技术
description: 对 HandlerThread 原理的简要分析。
---

`HandlerThread`是一个可以执行多个异步操作，而不需要创建多线程的类。`HandlerThread`继承于`Thread`，是`Thread`的子类。

如何使用`HandlerThread`：

```java
private Handler myHandler;
private void startHandlerThread() {
    HandlerThread handlerThread = new HandlerThread("handlerThread");
    handlerThread.start();
    myHandler = new Handler(handlerThread.getLooper()) {
        @Override
        public void handleMessage(Message msg) {
            //执行耗时操作
            switch (msg.what) {
                default:
                    break;
            }
        }
    };
}
```
如果我们要执行某一个耗时操作，就只需要通过`myHandler`调用`sendMessage()`方法可以了。
那么他是如何实现的呢？首先我们要知道它是一个继承于`Thread`的类，那么当调用`start()`方法时就会开启线程，然后执行`run()`方法，接下来看这个方法是如何被重写的：

```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

```
其实就是在子线程里创建一个`Looper`对象，然后调用`loop()`方法循环获取消息，那么为什么`handleMessage(Message msg)`会在子线程里执行呢，我们可以看到其实在构建`Handler`的时候通过`handlerThread.getLooper()`将子线程的`looper`传入了，理所当然`loop()`中`msg.target.dispatchMessage(msg);`也会在子线程中执行。

其实在`IntentService`内部就是用了`HandlerThread`来实现的，不妨来看看它的源码：
```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    /**
     * Creates an IntentService.  Invoked by your subclass's constructor.
     *
     * @param name Used to name the worker thread, important only for debugging.
     */
    public IntentService(String name) {
        super();
        mName = name;
    }

    /**
     * Sets intent redelivery preferences.  Usually called from the constructor
     * with your preferred semantics.
     *
     * <p>If enabled is true,
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_REDELIVER_INTENT}, so if this process dies before
     * {@link #onHandleIntent(Intent)} returns, the process will be restarted
     * and the intent redelivered.  If multiple Intents have been sent, only
     * the most recent one is guaranteed to be redelivered.
     *
     * <p>If enabled is false (the default),
     * {@link #onStartCommand(Intent, int, int)} will return
     * {@link Service#START_NOT_STICKY}, and if the process dies, the Intent
     * dies along with it.
     */
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    /**
     * Unless you provide binding for your service, you don't need to implement this
     * method, because the default implementation returns null.
     * @see android.app.Service#onBind
     */
    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
     * This method is invoked on the worker thread with a request to process.
     * Only one Intent is processed at a time, but the processing happens on a
     * worker thread that runs independently from other application logic.
     * So, if this code takes a long time, it will hold up other requests to
     * the same IntentService, but it will not hold up anything else.
     * When all requests have been handled, the IntentService stops itself,
     * so you should not call {@link #stopSelf}.
     *
     * @param intent The value passed to {@link
     *               android.content.Context#startService(Intent)}.
     *               This may be null if the service is being restarted after
     *               its process has gone away; see
     *               {@link android.app.Service#onStartCommand}
     *               for details.
     */
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```
我们可以看到在`onCreate()`方法里创建了一个命名为`thread`的`HandlerThread`实例，同时也创建了一个`ServiceHandler`的实例，并将从`thread`里获取到的`looper`作为初始化参数传入，在`onStart()`方法中调用了`mServiceHandler.sendMessage(msg);`，再看`ServiceHandler`中的`handleMessage(Message msg)`方法中调用了`onHandleIntent((Intent)msg.obj);`，即我们要实现的抽象方法，该方法在子线程中执行。

`HanderThread`的优点在于我们可以不用重复创建线程执行异步操作，缺点也很明显，就是它是队列的形式执行异步任务。

