`AsyncTask`是Android平台提供的一个多线程框架，它可以很简单的用于更新UI线程，你可以在后台执行操作，并将操作的结果Push到UI线程。同时你完全不需要去操作线程和Handler。`AsyncTask`与Java中的多线程框架不同，它只适合用于短小的任务（最多几秒钟）。为什么会有这样的要求？这就涉及到内存泄漏，后面我们会说。

<!--more-->

在使用`AsyncTask`前，我们需要实现`AsyncTask`，并定义三种类型，分别是`请求参数`、`进度参数`、`返回结果`，其中进度参数通常用来更新进度。在`AsyncTask`中执行异步任务的时候，`AsyncTask`会依次调用`onPreExecute`、`doInBackground`、 `onProgressUpdate`、`onPostExecute`这四个回调方法，我们可以在这些回调方法中执行相关操作。

我们首先说说`AsyncTask`的变迁历史，在`Android 1.6` 开始，`AsyncTask`是使用线程池同时执行多个任务，但是为了避免多线程给应用程序带来的错误，从`Android 3.0`开始，`AsyncTask`默认后台只开启一个线程，这里的改变就是将默认的`Executor`换做了`SerialExecutor`。

### 初始化

AsyncTask中含有几个重要的静态成员变量，会在类加载的时候进行初始化：

  - **THREAD_POOL_EXECUTOR**：`ThreadExecutor`实例，用于执行Task。
  - **sPoolWorkQueue**：`LinkedBlockingQueue<Runnable>`实例，用于存放`THREAD_POOL_EXECUTOR`需要执行的Runable。
  - **sDefaultExecutor**：`SerialExecutor`实例，将Task **线性** 提交给`ThreadExecutor`执行。
  - **sHandler**：主线程的Handler，用于在主线程回调结果、界面刷新。

同时还有两个成员变量：

```Java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            ...
            Result result = doInBackground(mParams);
            ...
            return postResult(result);
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
                ...
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

  - **mWorker**：`Callable`接口的实现，作为Task的执行过程，在这里主要是调用`doInBackground`，并获取`result`。
  - **mFuture**：`FutureTask`的子类，`FutureTask`是`RunnableFuture`的子类，它接受一个`Callable`。其实`FutureTask`就是包装了`Callable`，并且可以对`Callable`进行取消操作。

### Task执行

```Java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    ...
    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```

在上面的代码中我们可以看到`execute`方法默认使用`SerialExecutor`，当然我们也可以调用`executeOnExecutor`来使用自己的`Executor`。在执行Task之前会回调`onPreExecute`方法，然后通过`Executor`执行Task。

```Java
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

从上面的代码可以看出，`SerialExecutor`先将Task加入到一个队列中，然后通过`ThreadExecutor`来执行`mFuture`。在前面`mFuture`的初始化代码中，`mFuture`会回调`doInBackground`方法，然后调用`postResult`将运行结果发送到主线程。

> 由于中间通过`SerialExecutor`执行Task，`ThreadExecutor`只会创建一个线程。

```Java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

### 主线程调用

在AsyncTask的说明中我们可以看到：

```
There are a few threading rules that must be followed for this class to work properly:

  1. The AsyncTask class must be loaded on the UI thread. This is done automatically as of JELLY_BEAN.
  2. The task instance must be created on the UI thread.
  3. execute(Params...) must be invoked on the UI thread.
  4. Do not call onPreExecute(), onPostExecute(Result), doInBackground(Params...), onProgressUpdate(Progress...) manually.
  5. The task can be executed only once (an exception will be thrown if a second execution is attempted.)
```

对于第一点，在`JELLY_BEAN`版本前，`AsyncTask`的加载必须在主线程执行，在`JELLY_BEAN`时，在`ActivityThread`中进行`AsyncTask`的类加载。

```Java
//以下是 Android JELLY_BEAN 的代码

//ActivityThread.java
public static ActivityThread systemMain() {
    HardwareRenderer.disable(true);
    ActivityThread thread = new ActivityThread();
    thread.attach(true);

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    AsyncTask.init();

    ...
 }

//AsyncTask.java
public static void init() {
    //当前在ActivityThread中，即主线程中
    sHandler.getLooper();
}
```

对于第二点和第三点，与UI更新有关，我们可以在源码中看到，`onPreExecute`，`onProgressUpdate`等方法都注释有`@MainThread`。**如果我们不需要更新UI，可以在其他线程中创建AsyncTask的实例**。

### 内存泄漏

在本文开始的时候，我们就提到`AsyncTask`只适合 “短小” 的任务。如果`AsyncTask`是`Activity`的内部类，就会持有`Activity`的引用，`AsyncTask`中任务时间过长，可能导致`Activity`不能及时回收，从而导致内存泄漏。
