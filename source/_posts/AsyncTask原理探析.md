---
title: AsyncTask原理探析
date: 2019-10-13 19:17:12
tags: 
  - android
categories:
  - 技术
description: 对 AsyncTask 原理的简要分析。
---
`AsyncTask`是一个在安卓开发中帮助开发者更简便地创建异步任务的类。先来看简单看一下其用法：
```java
new AsyncTask<String,Integer,String>(){

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }

    @Override
    protected void onPostExecute(String s) {
        super.onPostExecute(s);
    }

    @Override
    protected String doInBackground(String... strings) {
        return null;
    }
}.execute("https://cn.bing.com/");
```
首先我们来看三个类型参数，第一个`Params`表示执行任务时传入参数类型，第二个`Progress`表示后台执行任务更新的进度类型，第三个`Result`表示后台任务执行结束返回结构的类型。
接下来我们来分析下`AsyncTask`初始化做了些什么。
```java
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);

    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```
主要做了三件事，第一创建`InternalHandler`类型的对象，这个`Handler`与主线程的`looper`绑定；第二创建`mWorker`对象，`WorkerRunnable`是一个`implements`了`Callable`接口的抽象类，这个`Callable`接口里定义了一个`call()`方法，可以看到这个重写的`call()`方法主要执行的就是异步操作，`doInBackground(mParams);`就在这里被调用；第三步创建`FutureTask`对象，`FutureTask`封装了`WorkerRunnable`中的`call()`方法。
以调用`execute()`入口，最后调用的是`executeOnExecutor()`方法。
```java
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
第一步会判断当前的任务执行状态，如果是正在执行中或者已经执行结束，那么会抛出异常，所以创建的`AsyncTask`无法多次执行`executed()`，如果是等待执行状态，那么将状态改为执行中；第二步执行`onPreExecute();`，也就是我们执行异步任务之前做的准备，这个方法在我们创建异步任务时可以重写添加自己的处理逻辑；第三步将传入的参数赋值给`mWorker`中的变量；第四步`exec.execute(mFuture);`
```java
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```
`exec`是`SerialExecutor`的实例，是`AsyncTask`中的静态变量，类加载的时候被初始化，被所有实例所共有，它其实是一个双端队列，保存着待执行的任务。ok，`mFuture`加入队列之后，如果没有正在执行的任务，即调用`scheduleNext();`
```java
protected synchronized void scheduleNext() {
    if ((mActive = mTasks.poll()) != null) {
        THREAD_POOL_EXECUTOR.execute(mActive);
    }
}
```
`THREAD_POOL_EXECUTOR`是静态代码块创建的一个线程池，这里就是把任务从队列中取出交给线程池执行，最终会调用`mWorker`的`call()`方法，`doInBackground()`执行完返回一个结果最终调用`postResult(result);`
```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

`getHandler()`获取到的是`InternalHandler`实例，接着看

```java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
接着调用`AsyncTask`中的`finish(result.mData[0]);`方法
```java
 private void finish(Result result) {
     if (isCancelled()) {
         onCancelled(result);
     } else {
         onPostExecute(result);
     }
     mStatus = Status.FINISHED;
 }
```
最后调用`onPostExecute(result);`返回执行结果。
我始终还想不明白 Google 工程师设计成串行的目的是为了什么？？？