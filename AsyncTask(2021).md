从构造函数开始看：因为第二个和第三个构造函数hide了，所以传入第三个构造函数时只能穿空的looper,从而 mHandler = getMainHandler()

```
  public AsyncTask() {
        this((Looper) null);
    }

    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }

	 /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     *
     * @hide
     */
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {//1
            public Result call() throws Exception {
            //当前任务被调用标志
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

        mFuture = new FutureTask<Result>(mWorker) {//2
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



```java
//1处的WorkerRunnable，实现了Callable
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
    
 //Callable接口代码
 public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

```

可见在1处的call方法里调用了doInBackground，并最终调用postResult，将结果传出去。

再看2处的代码
FutureTask实现了RunnableFuture,RunnableFuture<V> 继承了 Runnable, Future<V>

从上mWorker作为参数传递给了FutureTask



接下来看我们用AsyncTask时会手动调用的execute方法

```
  @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    
    @MainThread
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

 首先调用我们覆写的onPreExecute()，再将参数传给WorkerRunnable。

然后调用exec的execute方法并传入mFuture，这里的exec是execute中传入的sDefaultExecutor，它实际是串行的线程池SerialExecutor

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();    
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

  private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {//1
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

注释1处将mFuture加入到mTasks中，仅仅是加入，并没有执行run

只有当mActive=null时会开始执行scheduleNext，而所有runnable最终都会执行scheduleNext。

scheduleNext会执行mTasks队列的任务。即mFuture的run方法（r.run()）。而最终会调用mWorker的call方法。call方法里有  result = doInBackground(mParams)以及 postResult(result)方法。

上面的THREAD_POOL_EXECUTOR即

```java
  	private static final int CORE_POOL_SIZE = 1;
    private static final int MAXIMUM_POOL_SIZE = 20;
    private static final int BACKUP_POOL_SIZE = 5;
    private static final int KEEP_ALIVE_SECONDS = 3;

 	public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), sThreadFactory);
        threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

postResult代码如下：

```java
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

 /**
 
     * 这里的obtainMessage方法的源码如下
     * Same as {@link #obtainMessage()}, except that it also sets the what and obj members 
     * of the returned Message.
     * 
     * @param what Value to assign to the returned Message.what field.
     * @param obj Value to assign to the returned Message.obj field.
     * @return A Message from the global message pool.
     */
    @NonNull
    public final Message obtainMessage(int what, @Nullable Object obj) {
        return Message.obtain(this, what, obj);
    }

//AsyncTaskResult类代码：
//注意这里的mTask和SerialExecutor里的mTask不一样
 private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```

在postResult中创建message。得到handler并发送消息

*ps:message的创建方法：*

> //创建message
> //1、new的方式
> Message message1 = new Message();
> message1.what = 1;
>
> /*  以下三种方式的本质是一样的  */
>
> //2、Message.obtain的方式
> //从整个Messge池中返回一个新的Message实例,避免创建对象，从而减少内存的开销了。
> Message message2 = Message.obtain();
> message2.what = 1;
>
> //3、Handler.obtainMessage的方式
> //从整个Messge池中返回一个新的Message实例,避免创建对象，从而减少内存的开销了。
> Message message3 = new Handler().obtainMessage();
>
> //从整个Messge池中返回一个新的Message实例,避免创建对象，从而减少内存的开销了。
> Message message4 = new Handler().obtainMessage(1);
> //what=1,同message1，message2相同

getHandler()代码如下：

```java
 private Handler getHandler() {
        return mHandler;
    }
```

从第三个构造函数可以看到mHandler = getMainHandler()

```java
 private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
```

InternalHandler代码如下：

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

case MESSAGE_POST_RESULT里实际调用AsyncTask里的finish方法 :

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



**AsyncTask中有两个线程池（SerialExecutor和THREAD_POOL_EXECUTOR），和一个Handler (InternalHandler)，其中SerialExecutor用于任务的排队，而THREAD_POOL_EXECUTOR用于真正地执行任务。InternalHandler用于子线程与主线程的通信。**



**Q ** : 为什么AsyncTask的类必须在主线程中加载？

注意题目问的是类的加载，而非类对象的创建。

首先理解一下这两个发送message的方法。

> ```undefined
> Message msg = handler.obtainMessage(); 
> msg.arg1 = i; 
> msg.sendToTarget(); 
> 
> Message msg=new Message(); 
> msg.arg1=i; 
> handler.sendMessage(msg); 
> ```
>
> - 第一种写法是message 从handler 类获取，从而可以直接向该handler 对象发送消息;
> - 第二种写法是直接调用 handler 的发送消息方法发送消息。
>
> > 推荐第一种写法，此写法的 message 来自 MessagePool ,省去了创建对象申请内存的开销
>
>

**A ** :  因为mWoker里的postResult方法用的是上面的第一个方式。结合Handler相关知识，知道要向哪发消息，则应该在哪里创建Handler。这里的Handler实际是静态成员。由于静态成员会在类加载的时候进行初始化，因此变相要求AsyncTask的类必须在主线程中加载。

而这个AsyncTask的类在主线程中加载的过程在Android4.1及以上版本中已经被系统自动完成了。在ActivelyThread的main方法会调用AsyncTask的init方法。



**Q** : 为什么AsyncTask默认为串行执行的呢？

**A** : 由于一个进程内所有的AsyncTask都是使用的同一个线程池执行任务；如果同时有几个AsyncTask一起并行执行的话，恰好AysncTask的使用者在doInbackgroud里面访问了相同的资源，但是自己没有处理同步问题；那么就有可能导致灾难性的后果！

由于开发者通常不会意识到需要对他们创建的所有的AsyncTask对象里面的doInbackgroud做同步处理，因此，API的设计者为了避免这种无意中访问并发资源的问题，干脆把这个API设置为默认所有串行执行的了。

如果你明确知道自己需要并行处理任务，那么你需要使用executeOnExecutor(Executor exec,Params... params)这个函数来指定你用来执行任务的线程池



额外补充几点注意事项：

1.AsyncTask的对象必须在UI线程中创建，在源码里规定的

2.execute方法必须在UI线程中调用，在源码里被注解的

3.一个AsyncTask对象的execute方法只能调用一次，否则会报运行时异常