####  Looper.loop是一个死循环，拿不到需要处理的Message就会阻塞，那在UI线程中为什么不会导致ANR？

- 问题描述 

  - 在处理消息的时候使用了Looper.loop()方法，并且在该方法中进入了一个死循环，同时Looper.loop()方法是在主线程中调用的，那么为什么没有造成阻塞呢？

- ActivityThread中main方法 

  - ActivityThread类的注释上可以知道这个类管理着我们平常所说的主线程(UI线程) 

    - 首先 ActivityThread 并不是一个 Thread，就只是一个 final 类而已。我们常说的主线程就是从这个类的 main 方法开始，main 方法很简短

    ```
    public static final void main(String[] args) {
        ...
        //创建Looper和MessageQueue
        Looper.prepareMainLooper();
        ...
        //轮询器开始轮询
        Looper.loop();
        ...
    }
    复制代码
    ```

- Looper.loop()方法无限循环 

  - 看看Looper.loop()方法无限循环部分的代码

    ```
    while (true) {
       //取出消息队列的消息，可能会阻塞
       Message msg = queue.next(); // might block
       ...
       //解析消息，分发消息
       msg.target.dispatchMessage(msg);
       ...
    }
    复制代码
    ```

- 为什么这个死循环不会造成ANR异常呢？ 

  - 因为Android 的是由事件驱动的，looper.loop() 不断地接收事件、处理事件，每一个点击触摸或者说Activity的生命周期都是运行在 Looper.loop() 的控制之下，如果它停止了，应用也就停止了。只能是某一个消息或者说对消息的处理阻塞了 Looper.loop()，而不是 Looper.loop() 阻塞它。[技术博客大总结](https://github.com/yangchong211/YCBlogs)

- 处理消息handleMessage方法 

  - 如下所示 

    - 可以看见Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施。
    - 如果某个消息处理时间过长，比如你在onCreate(),onResume()里面处理耗时操作，那么下一次的消息比如用户的点击事件不能处理了，整个循环就会产生卡顿，时间一长就成了ANR。

    ```
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            break;
            case RELAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                ActivityClientRecord r = (ActivityClientRecord) msg.obj;
                handleRelaunchActivity(r);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            break;
            case PAUSE_ACTIVITY:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                handlePauseActivity((IBinder) msg.obj, false, (msg.arg1 & 1) != 0, msg.arg2, (msg.arg1 & 2) != 0);
                maybeSnapshot();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case PAUSE_ACTIVITY_FINISHING:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
                handlePauseActivity((IBinder) msg.obj, true, (msg.arg1 & 1) != 0, msg.arg2, (msg.arg1 & 1) != 0);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            ...........
        }
    }
    复制代码
    ```

- loop的循环消耗性能吗？ 

  - 主线程Looper从消息队列读取消息，当读完所有消息时，主线程阻塞。子线程往消息队列发送消息，并且往管道文件写数据，主线程即被唤醒，从管道文件读取数据，主线程被唤醒只是为了读取消息，当消息读取完毕，再次睡眠。因此loop的循环并不会对CPU性能有过多的消耗。
  - 简单的来说：ActivityThread的main方法主要就是做消息循环，一旦退出消息循环，那么你的程序也就可以退出了。