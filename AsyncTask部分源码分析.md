### AsyncTask部分源码分析

先从构造方法开始理解源码

```
public AsyncTask(@Nullable Looper callbackLooper) {

//初始化mHandler
//  如果callbackLooper没有自行设置，会直接使用UI线程的Handler
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
        

/*
private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
*/

/* 
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
                    //在接收到MESSAGE_POST_RESULT消息后会调用AsyncTask的finish方法
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
               
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
    */
    
    /*private void finish(Result result) {
    
       如果AsyncTask任务被取消了，则执行onCancelled方法，否则就调用onPostExecute方法。正是通过 onPostExecute方法，我们才能够得到异步任务执行后的结果
               
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }*/
    
/*
初始化WorkerRunnable
WorkerRunnable实现了Callable接口，并实现了它的call方法，在call方法中调用了doInBackground（mParams）来处理任务并得到结果，并最终调用postResult将结果投递出去。
*/
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {

// 当前任务是否被调用 
//private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
 // 添加线程的调用标识
            mTaskInvoked.set(true);
            Result result = null;
            try {
// 设置线程的优先级
               Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                
//调用doInBackground(mParams)方法，doInBackground(mParams)由开发者来实现
                result = doInBackground(mParams);
////将进程中未执行的命令,一并送往CPU处理
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
//如果运行异常,设置取消的标志
                mCancelled.set(true);
                throw tr;
            } finally {
//发送结果
                postResult(result);
            }
            return result;
        }
    };
    /*
    postResult中出现了我们熟悉的异步消息机制，AsyncTaskResult就是一个简单的携带参数的对象，最终通过
postResult将结果投递给UI线程。
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}*/
/*
RunnableFuture继承了Runnable接口和Future接口，而FutureTask实现了RunnableFuture接口。
所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。
在这里WorkerRunnable作为参数传递给了FutureTask。这两个变量会暂时保存在内存中，
*/
    mFuture = new FutureTask<Result>(mWorker) {

//done（）简介：FutureTask内的Callable执行完后的调用方法
// 作用：复查任务的调用、将未被调用的任务的结果通过InternalHandler传递到UI线程
      
        @Override
        protected void done() {
            try {
 // 在执行完任务后检查,将没被调用的Result也一并发出
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
 //若发生异常,则将发出null
                postResultIfNotInvoked(null);
            }
        }
    };
}
```





```
 private void postResultIfNotInvoked()(Result result) {
        // 取得任务标记
        final boolean wasTaskInvoked = mTaskInvoked.get();

        // 若任务无被执行,将未被调用的任务的结果通过InternalHandler传递到UI线程
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```

对于AsyncTask，我们最先调用的就是它的`execute()`方法，所以继续看相关代码

```
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

```
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {

        //  判断 AsyncTask 当前的执行状态
        // PENDING = 初始化状态
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

  //将AsyncTask状态设置为RUNNING状态
    mStatus = Status.RUNNING;

 // 主线程初始化工作,由开发者实现
    onPreExecute();

 // 添加参数到任务中
//将 AsyncTask的参数传给WorkerRunnable
    mWorker.mParams = params;

/*执行任务
这里exec是传进来的参数sDefaultExecutor，它是一个串行的线程池 SerialExecutor
AsyncTask中所有Task共享一个SerialExecutor实例，保证所有任务都是串行
从前面我们知道WorkerRunnable作为参数传递给了FutureTask，因此，参数被封装到FutureTask中。接下来会调用exec的execute方法，并将mFuture也就是前面讲到的FutureTask传进去。
*/
    exec.execute(mFuture);

    return this;
}
```

```
private static class SerialExecutor implements Executor {
        // SerialExecutor是静态内部类，即所有实例化的AsyncTask对象公有
	 // SerialExecutor 内部维持了1个双向队列；
        // 容量根据元素数量调节
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        // execute（）被同步锁synchronized修饰
        // 即说明：通过锁使得该队列保证AsyncTask中的任务是串行执行的
        // 即 多个任务需一个个加到该队列中；然后 执行完队列头部的再执行下一个，以此类推
        public synchronized void execute(final Runnable r) {
            // 将实例化后的FutureTask类 的实例对象传入
            // 即相当于：向队列中加入一个新的任务
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            // 若当前无任务执行，则去队列中取出1个执行
            if (mActive == null) {
                scheduleNext();
            }
        }
       
        protected synchronized void scheduleNext() {
            // 取出队列头部任务
            if ((mActive = mTasks.poll()) != null) {

                //  执行取出的队列头部任务
                // 即 调用执行任务线程池类（THREAD_POOL_EXECUTOR）
                THREAD_POOL_EXECUTOR.execute(mActive);
                
            }
        }
    }

```

### **为什么AsyncTask默认把它设计为串行执行的呢？**

由于一个进程内所有的AsyncTask都是使用的同一个线程池执行任务；如果同时有几个AsyncTask一起并行执行的话，恰好AysncTask的使用者在doInbackgroud里面访问了相同的资源，但是自己没有处理同步问题；那么就有可能导致灾难性的后果！

由于开发者通常不会意识到需要对他们创建的所有的AsyncTask对象里面的doInbackgroud做同步处理，因此，API的设计者为了避免这种无意中访问并发资源的问题，干脆把这个API设置为默认所有串行执行的了。

如果你明确知道自己需要并行处理任务，那么你需要使用executeOnExecutor(Executor exec,Params... params)这个函数来指定你用来执行任务的线程池

 