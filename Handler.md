### 为什么需要Handler

------

Handler的主要作用是将一个任务切换到某个指定的指定的线程中执行。而Handler常常可以用于在子线程更新UI。

系统为什么不允许在子线程更新UI呢？这是因为Android的UI控件不是线程安全的，如果在多线程中并发访问可能会导致UI控件处于不可预期的状态。那么为什么系统不对UI控件的访问加上锁机制呢？缺点有两个：首先加上锁机制会让UI访问的逻辑变得复杂；其次，锁机制会降低UI的访问效率，因为锁机制会阻塞某些线程的执行。鉴于这两个缺点，最简单且高效的方法就是采用单线程模型来处理UI操作，对于开发者来说也不是很麻烦，只是需要通过Handler切换一下UI访问的执行线程即可。

### 概述

------

Handler运行需要底层的MessageQueue和Looper支撑。其中MessageQueue采用的是单链表的结构，Looper可以叫做消息循环。由于MessageQueue只是一个消息存储单元，不能去处理消息，而Looper就是专门来处理消息的，Looper会以无限循环的形式去查找是否有新消息，如果有的话，就处理，否则就一直等待着。

当Looper发现有新消息到来时，就会处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法就会被调用。注意Looper是运行在创建Handler所在的线程中的，这样一来Handler中的业务逻辑就被切换到创建Handler所在的线程中去执行了。

我们知道，Handler创建的时候会采用 ***当前线程*** 的Looper来构造消息循环系统，需要注意的是，线程默认是没有Looper的，如果需要使用Handler就必须为线程创建Looper，因为默认的UI主线程，也就是ActivityThread，ActivityThread被创建的时候就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。



#### Message

> 定义：是线程间通讯的数据单元，包含着描述信息及任意数据对象，发送到 Handler。

在实际使用中，我们在工作线程中通过 Handler.sendMessage(Message)，将携带数据的 Message 信息发送给 Handler，然后再由 Handler 处理，根据不同的信息，通知主线程作出相对应的 UI 工作。



#### MessageQueue 消息队列

> 定义：用来存储 Message 的数据队列。

主要包含两个操作：插入和读取。读取操作本身会伴随着删除操作。插入对应的方法为enqueueMessage，读取对应的方法为next。next方法是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next会返回这条消息并将其从单链表中移除。



#### Looper 消息循环器

> 定义：用于为线程执行消息循环的一个类。是 MessageQueue 与 Handler 之间的通讯媒介。

会不停地从MessageQueue中查看是否有新消息，如果有新消息就立即处理，否则就一直阻塞在那里。

在子线程中，如果手动为其创建了Looper，那么在所有的事情完成以后应该调用quit方法来终止消息循环，否则这个子线程就会一直处于等待状态。而**如果退出Looper以后，这个线程就会立刻终止 **。因此建议在不需要的时候终止Looper



#### 关系

![20200104121245311.png](https://i.loli.net/2021/03/27/n4dvK1o3k8MgyPH.png)

存储在 **MessageQueue** 中的 **Message** 被 **Looper** 循环分发到指定的 **Handler** 中进行处理

- 一个 Thread 可以有多个 Handler。
- 但一个 Thread 只能有一个 Looper，即在该线程调用Looper.prepare()创建的Looper
- 一个 Handler 只能关联一个 Looper 对象，即创建Handler线程的Looper
- 反之，一个 Looper 可以被多个 Handler 所关联
- 一个Looper创建一个MessageQueue



### Handler的构造方法

------

```java
@UnsupportedAppUsage
final Looper mLooper;
final MessageQueue mQueue;
@UnsupportedAppUsage
final Callback mCallback;
final boolean mAsynchronous;

public Handler() {
    this(null, false);
}

 public Handler(@Nullable Callback callback) {
        this(callback, false);
    }

public Handler(@NonNull Looper looper) {
        this(looper, null, false);
    }

 public Handler(@NonNull Looper looper, @Nullable Callback callback) {
        this(looper, callback, false);
    }

 @UnsupportedAppUsage
    public Handler(boolean async) {
        this(null, async);
    }

public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

### Handler的使用

------

#### 1.新建Handler子类（内部类）

```java
public class MainActivity extends AppCompatActivity {
    
    public TextView mTextView;
    public Handler mHandler;

    // 步骤1：（自定义）新创建Handler子类(继承Handler类) & 复写handleMessage（）方法
   static class Mhandler extends Handler {

        // 通过复写handlerMessage() 从而确定更新UI的操作
        @Override
        public void handleMessage(Message msg) {
            // 根据不同线程发送过来的消息，执行不同的UI操作
            // 根据 Message对象的what属性 标识不同的消息
            switch (msg.what) {
                case 1:
                    mTextView.setText("执行了线程1的UI操作");
                    break;
                case 2:
                    mTextView.setText("执行了线程2的UI操作");
                    break;
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.show);

        // 步骤2：在主线程中创建Handler实例
        mHandler = new Mhandler();
       
        
        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                 // 步骤3：创建所需的消息对象
                 Message msg = Message.obtain();
                 msg.what = 1; // 消息标识
                 msg.obj = "A"; // 消息内存存放

                 // 步骤4：在工作线程中 通过Handler发送消息到消息队列中
                 mHandler.sendMessage(msg);
            }
        }.start();
        // 步骤5：开启工作线程（同时启动了Handler）

        // 此处用2个工作线程展示
        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(6000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 通过sendMessage（）发送
                 // a. 定义要发送的消息
                 Message msg = Message.obtain();
                 msg.what = 2; //消息的标识
                 msg.obj = "B"; // 消息的存放
                 // b. 通过Handler发送消息到其绑定的消息队列
                 mHandler.sendMessage(msg);
            }
        }.start();

    }
}

```

#### 2.匿名内部类

```java
public class MainActivity extends AppCompatActivity {
    
    public TextView mTextView;
    public Handler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.show);

        // 步骤1：在主线程中 通过匿名内部类 创建Handler类对象
        mHandler = new Handler(){
            // 通过复写handlerMessage()从而确定更新UI的操作
            @Override
            public void handleMessage(Message msg) {
                // 根据不同线程发送过来的消息，执行不同的UI操作
                switch (msg.what) {
                    case 1:
                        mTextView.setText("执行了线程1的UI操作");
                        break;
                }
            }
        };
     
        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                 // 步骤3：创建所需的消息对象
                 Message msg = Message.obtain();
                 msg.what = 1; // 消息标识
                 msg.obj = "A"; // 消息内存存放

                 // 步骤4：在工作线程中 通过Handler发送消息到消息队列中
                 mHandler.sendMessage(msg);
            }
        }.start();
        // 步骤5：开启工作线程（同时启动了Handler）

      
    }

}

```

#### 3.Handler.post（Runnable）

```java
public class MainActivity extends AppCompatActivity {
    
    public TextView mTextView;
    public Handler mHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.show);

        // 步骤1：在主线程中创建Handler实例
        mHandler = new Handler();

        // 步骤2：在工作线程中 发送消息到消息队列中 & 指定操作UI内容
        new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 通过psot（）发送，需传入1个Runnable对象
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        // 指定操作UI内容
                        mTextView.setText("执行了线程1的UI操作");
                    }

                });
            }
        }.start();
        // 步骤3：开启工作线程（同时启动了Handler）


    }

}

```



先看一下sendMessage的源码分析一下执行过程

```java
 public final boolean sendMessage(@NonNull Message msg) {
        return sendMessageDelayed(msg, 0);
    }

 public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

   public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

 private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```

可以发现，Handler发送消息的过程仅仅是向消息队列插入了一条消息（调用MessageQueue的enqueueMessage方法）。MessageQueue的next方法就会返回这条消息给Looper，Looper收到消息后就开始处理，最终交由Handler处理。即Handler的dispatchMessage方法被调用。

```java
  public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
            
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

Handler处理消息的过程如下：

首先，检查Message的callback（msg.callback）是否为null，不为null就通过handleCallback来处理消息。这里的msg.callback即为Handler的post方法传进来的Runnable对象。

```java
 public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

    @UnsupportedAppUsage
    private static Message getPostMessage(Runnable r, Object token) {
        Message m = Message.obtain();
        m.obj = token;
        m.callback = r;
        return m;
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }
```



其次检查mCallback是否为null。不为null就调用mCallback的HandleMessage方法来处理消息。Callback是个接口，定义如下：

```java
public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        boolean handleMessage(@NonNull Message msg);
    }
    
```

通过Callback可以通过Handler handler=new Handler(callback)来创建Handler对象。其作用在于可以用来创建Handler实例而不需要派生Handler的子类

最后调用Handler的handleMessage方法来处理消息。handleMessage方法是Handler子类需要实现的。



### 子线程创建Handler

------

```java
class CustomChildThread extends Thread {
	@Override
    public void run() {
    	//为当前线程创建一个 Looper 对象
        Looper.prepare();
        
		//在子线程中创建一个 Handler 对象
        Handler handler = new Handler() {
            public void handleMessage(Message msg) {
                // 在这里处理传入的消息
            }
        };
        //开始消息循环
        Looper.loop();
    }
}
```



//TODO:Handler构造函数中有一个async参数，与同步，异步消息有关。涉及到同步屏障。网上的说法：`屏障消息` 的出现，他的作用是：**忽略所有的同步消息，返回异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息。**`View绘制` 相关的知识点中就有**同步屏障** 的使用：https://dandanlove.blog.csdn.net/article/details/78775166 挖个坑以后填qwq