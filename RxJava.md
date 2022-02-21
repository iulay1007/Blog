# RxJava



### 创建被观察者

#### Observable

```java
public abstract class Observable<T> implements ObservableSource<T> {...}
```

Observable是一个抽象类，实现了ObservableSource接口

#### ObservableSource

```java
public interface ObservableSource<T> {

    /**
     * Subscribes the given Observer to this ObservableSource instance.
     * @param observer the Observer, not null
     * @throws NullPointerException if {@code observer} is null
     */
    void subscribe(@NonNull Observer<? super T> observer);
}
```

ObservableSource接口中只有一个subscribe方法。



#### Observable#create()

```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

#### observableOnSubscribe接口

```java
public interface ObservableOnSubscribe<T> {

    void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
}
```

将参数source传入ObservableCreate的构造方法中创建ObservableCreate类型对象，并将该对象传入RxJavaPlugins.onAssembly（Observable<T> source）中



#### ObservableCreate

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }
...
}
```

可以看出，ObservableCreate的构造函数仅仅是将传入的参数赋值给成员变量进行保存。



#### RxJavaPlugins#onAssembly()

```java
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    //Calls the associated hook function
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```

可见，在一般情况下，这个方法仅仅是将参数返回。



**至此，我们已经知道了创建被观察者的代码流程。在create方法中，传入的是ObservableOnSubscribe接口的实现，该接口只有一个方法。而返回的是Observable的子类，实际上也就是ObservableCreate对象。**

**可以发现，在这里使用了适配器模式。将ObservableOnSubscribe的实现传入ObservableCreate类中，使其转换为目标类，且同时增添了功能。单从增加功能这一个角度看，很像装饰模式，但是装饰模式的前提是不改变接口。而从ObservableOnSubscribe接口到ObservableCreate类的转换已经将接口改变了，所以不符合装饰模式的定义，而是更加符合适配器模式，即在不改变原先代码的前提下，使得现有的代码适配新的接口。**



### 订阅过程

具体逻辑是：被观察者调用subscribe方法订阅，这时应先传入观察者。然后被观察者会调用观察者的onSubscribe方法通知观察者订阅成功，同时，被观察者也会调用我们自己实现的ObservableOnSubscribe接口的subscribe方法，这个方法持有观察者的引用，可以对观察者进行操作。

具体实现如下：

#### Observable#subscribe()

```java
 public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);//1

            ObjectHelper.requireNonNull(observer, "The RxJavaPlugins.onSubscribe hook returned a null Observer. Please change the handler provided to RxJavaPlugins.setOnObservableSubscribe for invalid null returns. Further reading: https://github.com/ReactiveX/RxJava/wiki/Plugins");

            subscribeActual(observer);//2
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



注释1处

#### RxJavaPlugins#onSubscribe()

```java
 @NonNull
    public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {
        BiFunction<? super Observable, ? super Observer, ? extends Observer> f = onObservableSubscribe;
        if (f != null) {
            return apply(f, source, observer);
        }
        return observer;
    }
```

应用了hook技术，在一般情况下，仅仅是将传入的参数返回。



注释2处

#### Observable#subscribeActual()

这是一个抽象方法。具体的实现在Observable的子类，在本例中具体的实现在ObservableCreate类对象中。

```java
 protected abstract void subscribeActual(Observer<? super T> observer);
```



#### ObservableCreate#subscribeActual()

```java
  @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);//1
        observer.onSubscribe(parent);//2

        try {
            source.subscribe(parent);//3
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```

注释1处

#### CreateEmitter

```java
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {

        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

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

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public boolean tryOnError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
                return true;
            }
            return false;
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }

        @Override
        public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }

        @Override
        public String toString() {
            return String.format("%s{%s}", getClass().getSimpleName(), super.toString());
        }
    }
```

CreateEmitter继承了AtomicReference<Disposable>，所以保证了Disposable的一致性。

> Atomic家族主要是保证多线程环境下的原子性，相比synchronized而言更加轻量级。比较常用的是AtomicInteger，作用是对Integer类型操作的封装，而AtomicReference作用是对普通对象的封装。

且CreateEmitter实现了ObservableEmitter<T>和 Disposable接口

CreateEmitter<T> parent = new CreateEmitter<T>(observer);将observer传入CreateEmitter类，为其扩展增加了事件发射的功能，但依然改变了它的接口，所以也是适配器模式。

#### Disposable

```java
public interface Disposable {
    /**
     * Dispose the resource, the operation should be idempotent.
     */
    void dispose();

    /**
     * Returns true if this resource has been disposed.
     * @return true if this resource has been disposed
     */
    boolean isDisposed();
}
```



注释2处

**observer.onSubscribe(parent);**

observer是Observable的subscribe调用方法subscribeActual传进来的观察者参数，实际上是我们自定义的观察者对象。

observer的onSubscribe也是我们自定义observer时需要实现的。

```
    void onSubscribe(@NonNull Disposable d);
```

其参数为Disposable类型，CreateEmitter实现了Disposable接口，所以可以进一步看出，我们向CreateEmitter构造函数传入observer，使其可以适配Disposable接口。

该方法是在被观察者的subscribe方法里的观察者的回调方法，可以用来通知观察者订阅成功。CreateEmitter类型的参数parent，是对observer功能的扩展，使其具备一些如中断处理的功能，这样就可以在观察者收到订阅成功的消息后，进行进一步的操作。



注释3处

 **source.subscribe(parent)**

source也就是ObservableCreate类保存的observableOnSubscribe接口实例。

**observableOnSubscribe#subscribe**

```
void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
```

这个方法中需要传入ObservableEmitter类型的参数，即实现了ObservableEmitter的CreateEmitter类型对象parent。

observableOnSubscribe的subscribe方法也是我们自己定义的，通常会在其中调用emitter.onNext（）和 emitter.onComplete（）方法。

> 注意区分subscribe（）方法和onSubscribe（）方法，前者属于被观察者，后者属于观察者。

emitter.onNext（）方法具体的实现在CreateEmitter类中。

这里的observer就是我们在构造CreateEmitter对象时传入的我们自定义的观察者。它的onNext（）和onComplete（）方法是我们自定义观察者时实现的。

```java
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

```java
public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}
```

可以看到这两个方法中都有isDisposed()的判断

#### CreateEmitter#isDisposed()

```java
public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
```

#### AtomicReference#get()

```java
private volatile V value;

public final V get() {
        return value;
    }
```

由于我们没有设置value，所以返回值为null

#### DisposableHelper#isDisposed()

```java
public enum DisposableHelper implements Disposable {
    /**
     * The singleton instance representing a terminal, disposed state, don't leak it.
     */
    DISPOSED
    ;

    /**
     * Checks if the given Disposable is the common {@link #DISPOSED} enum value.
     * @param d the disposable to check
     * @return true if d is {@link #DISPOSED}
     */
    public static boolean isDisposed(Disposable d) {
        return d == DISPOSED;
    }

```

此例中d == null,所以默认返回false。

isDisposed（）方法是用来判断当前事件是否被切断，在未进行任何设置的情况下，事件默认是不会被切断的。

在CreateEmitter类中，onNext()和onComplete()方法会先判断事件是否被切断，如果被切断了，就不会继续执行方法中的代码。如果事件没有被切断，onNext()和onComplete()方法会正常执行，onError()和onComplete()方法最后会执行dispose()方法。这个方法会将事件切断。

#### CreateEmitter#dispose()

```java
public void dispose() {
            DisposableHelper.dispose(this);
        }
```

#### DisposableHelper#dispose()

```java
public static boolean dispose(AtomicReference<Disposable> field) {
        Disposable current = field.get();
        Disposable d = DISPOSED;
        if (current != d) {
            current = field.getAndSet(d);
            if (current != d) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
        return false;
    }
```

#### AtomicReference#getAndSet()

```java
public final V getAndSet(V newValue) {
    return (V)unsafe.getAndSetObject(this, valueOffset, newValue);
}
```

AtomicReference的getAndSet方法通过调用unsafe类的getAndSetObject的CAS操作，保证了操作的原子性，解决了并发操作的读写问题。



### 线程切换

线程切换主要分为subscribeOn()和observeOn()方法

#### Observable#subscribeOn()

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));//1
}
```

如果对上面讲过的内容有印象，可以发现，这个方法和Observable的create方法的结构和形式十分相似。

这里需要传入Scheduler类型的参数。我们通常通过Schedulers类来获取Scheduler类型的子类。

常见的Scheduler类型如下

| Scheduler类型        | 使用方式                         | 含义            | 使用场景                                         |
| -------------------- | -------------------------------- | --------------- | ------------------------------------------------ |
| IoScheduler          | `Schedulers.io()`                | io操作线程      | 读写SD卡文件，查询数据库，访问网络等IO密集型操作 |
| NewThreadScheduler   | `Schedulers.newThread()`         | 创建新线程      | 耗时操作等                                       |
| SingleScheduler      | `Schedulers.single()`            | 单例线程        | 只需一个单例线程时                               |
| ComputationScheduler | `Schedulers.computation()`       | CPU计算操作线程 | 图片压缩取样、xml,json解析等CPU密集型计算        |
| TrampolineScheduler  | `Schedulers.trampoline()`        | 当前线程        | 需要在当前线程立即执行任务时                     |
| HandlerScheduler     | `AndroidSchedulers.mainThread()` | Android主线程   | 更新UI等                                         |

Schedulers.io()是我们常用的调度器类型，所以接下来看一下它的具体实现  

#### Schedulers#io()

```java
 public static Scheduler io() {
        return RxJavaPlugins.onIoScheduler(IO);
    }
```

#### Schedulers#IO

```java
static {
    ...
    IO = RxJavaPlugins.initIoScheduler(new IOTask())；
}
```

#### Schedulers#IOTask

```java
static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        return IoHolder.DEFAULT;
    }
}
```

#### Schedulers#IoHolder.DEFAULT

```java
static final class IoHolder {
    static final Scheduler DEFAULT = new IoScheduler();
}
```

RxJavaPlugins主要是执行一些hook操作，可以忽视。

所以Schedulers.io()实际是返回了一个IoScheduler类型的对象。

注释1处

**new ObservableSubscribeOn<T>(this, scheduler)**

#### ObservableSubscribeOn

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
...}
```

在Observable的subscribeOn方法中，将其自身传入ObservableSubscribeOn方法作为第一个参数。因为Observable实现了ObservableSource接口。

**super(source)**是调用父类构造函数，将source保存在父类的成员变量中。

可以看到这个ObservableSubscribeOn类存储了调度器的类型，以及ObservableSource接口的实例。

最开始有讲过RxJavaPlugins.onAssembly()方法，它需要接收一个Observable类型的对象。而ObservableSubscribeOn类也是Observable的子类，所以可以将ObservableSubscribeOn类型的对象传入RxJavaPlugins.onAssembly()方法中。一般情况下，RxJavaPlugins.onAssembly()方法只是将传入的参数返回。

Observable.subscribeOn()方法返回的是一个ObservableSubscribeOn类型的对象。它也是Observable的子类。

**所以，我们可以用它调用subscribe()订阅方法。它的具体实现在Observable类中。之前已经讲过。**

但与之前不同的是，这个方法中调用的这个方法：***subscribeActual(observer);***，它的具体实现要交由子类完成，而在这里，实际是调用了ObservableSubscribeOn类的该方法。

#### ObservableSubscribeOn#subscribeActual()

```java
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);//1

    observer.onSubscribe(parent);//2

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));//3
}
```

SubscribeOnObserver是ObservableSubscribeOn的内部类

#### SubscribeOnObserver

```java
static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

    private static final long serialVersionUID = 8094547886072529208L;
    final Observer<? super T> downstream;

    final AtomicReference<Disposable> upstream;

    SubscribeOnObserver(Observer<? super T> downstream) {
        this.downstream = downstream;
        this.upstream = new AtomicReference<Disposable>();
    }

    @Override
    public void onSubscribe(Disposable d) {
        DisposableHelper.setOnce(this.upstream, d);
    }

    @Override
    public void onNext(T t) {
        downstream.onNext(t);
    }

    @Override
    public void onError(Throwable t) {
        downstream.onError(t);
    }

    @Override
    public void onComplete() {
        downstream.onComplete();
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(upstream);
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }

    void setDisposable(Disposable d) {
        DisposableHelper.setOnce(this, d);
    }
}
```

可以看出，这个类和CreateEmitter类形式很相似。但最大的区别是，这个类具有AtomicReference<Disposable>类型名为upstream的成员变量。而CreateEmitter类是不具有的。因为正常情况下，调用CreateEmitter类对象的地方，便是最上游。

注释2

**observer.onSubscribe(parent)**

依然是一个回调方法。可以通知观察者订阅成功



注释3

**parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));**

这里的逻辑较为复杂，先给出一个结论，它最后会执行SubscribeTask类的run()方法。

```java
public void run() {
            source.subscribe(parent);
        }
```

要理解 source.subscribe(parent)，先要理解下面具体流程的逻辑图：

![未命名文件 (22).png](https://i.loli.net/2021/11/20/bkgAWwvBIMaZKUe.png)

注意区分，ObservableSubscribeOn的成员变量source指的是ObservableCreate对象，而ObservableCreate对象也有一个成员变量source，这个source指的是Observable在create时传进来的我们自定义的ObservableOnSubscribe。

ObservableSubscribeOn和ObservableCreate都没有重写subscribe方法，所以我们在调用subscribe时，其实会调用从Observable类继承来的subscribe方法。

这样看来，其实有点像递归操作，在一个方法中调用它本身。

但这个递归应该是有出口的，否则它就是一个死循环。那它的出口是什么呢？其实就是当source为我们实例的ObservableOnSubscribe接口。因为ObservableOnSubscribe中也有subscribe方法，我们无需再去调用Observable的subscribe方法，而是调用我们自定义的subscribe方法。

所以图中的1处，还会调用Observable的subscribe方法，直到碰到source为ObservableOnSubscribe时，便可以不再调用Observable的subscribe方法，即图中的2处。



ObservableSubscribeOn中持有Scheduler类型的成员变量，它将会在subscribeActual()方法的最后执行线程的切换操作。即source.subscribe(parent);会在Scheduler指定的线程中执行。这就是为什么subscribeOn()可以改变被观察者发送消息的线程。



#### 若我们多次调用subscribeOn()方法会出现什么情况？

由于每一个ObservableSubscribeOn中持有Scheduler类型的成员变量，所以它的subscribeActual()方法会根据Scheduler进行相应的线程切换。

图中的圆圈1，2表示线程1，2。

最外层的ObservableSubscribeOn（即拥有线程2的对象）执行subscribe方法时，会将线程切换到线程2，然后继续调用倒数第二层ObservableSubscribeOn方法，依次类推，它也会执行subscribe方法时，会将线程切换到线程1。然后继续执行倒数第三层的ObservableCreate的subscribe方法。但ObservableCreate不持有Scheduler类对象，也不会对线程进行切换，所以，此时的线程为线程1。

所以多次调用subscribeOn()方法，只有第一个subscribeOn()会对被观察者造成影响，其他的subscribeOn()也会造成线程切换，只是使用者感受不到。

![未命名1.png](https://i.loli.net/2021/11/20/3mLhAu7XUQwEvnY.png)



**subscribeOn()只影响其最近上游所在的线程。**

**它主要影响的是自定义被观察者的subscribe方法。**



#### doOnSubscribe()

doOnSubscribe()方法执行所在线程由其后面的subscribeOn()来指定；若没有subscribeOn()指定，则执行的线程和本身所在线程一致。

它和subscribeOn方法执行逻辑很像，只是它不会切换线程。

```java
public final Observable<T> doOnSubscribe(Consumer<? super Disposable> onSubscribe) {
    return doOnLifecycle(onSubscribe, Functions.EMPTY_ACTION);
}
```

#### Observable#observeOn()

通常用第一个方法，向其中传入调度器

```java
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}

 public final Observable<T> observeOn(Scheduler scheduler, boolean delayError) {
        return observeOn(scheduler, delayError, bufferSize());
    }

public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
    }
```

可以看出，最终会将其传入ObservableObserveOn中保存



#### ObservableObserveOn#subscribe

它的subscribe逻辑和之前提到的ObservableSubscribeOn的subscribe逻辑很像。都会先调用Observable的subscribe方法。而对于subscribeActual方法，不同Observable子类有不同逻辑。



#### ObservableObserveOn#subscribeActual

```java
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```

 **source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));**

主要是将observer进行进一步的传递与保存。

然后就是调用source（在这里也就是ObservableCreate）的subscribe方法。这个流程和前面讲过的一样。继续将observer进行进一步的传递与保存。

最终会调用自定义被观察者的subscribe方法。在这个方法中，假设我们调用了observer的onNext()方法。将会对之前进行层层包裹的observer进行调用。

首先，将会调用CreateEmitter的onNext()方法

#### CreateEmitter#onNext()

```java
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

正常情况下，会调用它保存的observer的onNext(),在此处是ObserveOnObserver的onNext()

#### ObserveOnObserver#onNext()

```java
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

这个方法较为复杂。结果是它会先切换线程，然后调用它保存的observer的onNext()。此处也就是调用我们自定义的observer的onNext()。

这个过程的具体流程图如下：

![2 (1).png](https://i.loli.net/2021/11/22/EqNmyAlTjiwXrk9.png)



#### 若我们多次调用observeOn()方法会出现什么情况？

相比一次调用observeOn()，多次调用只是进行了更多的线程切换，但只有离自定义的observe最近的会对它造成影响。

![2 (2).png](https://i.loli.net/2021/11/23/VZKAp5QGOInuCyT.png)

**subscribeOn 作用于该操作符之前的 Observable 的创建操符作以及 doOnSubscribe 操作符** ，换句话说就是 **doOnSubscribe 以及 Observable 的创建操作符总是被其之后最近的 subscribeOn 控制**



## Retrofit与RxJava

RxJava和Retrofit的组合是安卓开发里很常用的网络请求方法。

基本用法如下：

```java
Api service = retrofit.create(Api.class);
Observable<Msg> observable = service.getMsg();
```

```java
observable.subscribeOn(Schedulers.io()) 
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Observer<Msg>() { 
            @Override
            public void onCompleted() {
            }
            @Override
            public void onError(Throwable e) {
            }
            @Override
            public void onNext(Msg msg) {
            }
        });
```

这里的observable是CallExecuteObservable类型的。作为Observable的子类，我们依旧关注它对subscribeActual方法的实现。

### CallExecuteObservable#subscribeActual()

```java
protected void subscribeActual(Observer<? super Response<T>> observer) {
  // Since Call is a one-shot type, clone it for each new observer.
  Call<T> call = originalCall.clone();
  CallDisposable disposable = new CallDisposable(call);
  observer.onSubscribe(disposable);
  if (disposable.isDisposed()) {
    return;
  }

  boolean terminated = false;
  try {
    Response<T> response = call.execute();//1
    if (!disposable.isDisposed()) {
      observer.onNext(response);
    }
    if (!disposable.isDisposed()) {
      terminated = true;
      observer.onComplete();
    }
  } catch (Throwable t) {
    Exceptions.throwIfFatal(t);
    if (terminated) {
      RxJavaPlugins.onError(t);
    } else if (!disposable.isDisposed()) {
      try {
        observer.onError(t);
      } catch (Throwable inner) {
        Exceptions.throwIfFatal(inner);
        RxJavaPlugins.onError(new CompositeException(t, inner));
      }
    }
  }
}
```

#### 注释1

**Response<T> response = call.execute();**

又见到了熟悉的call.execute()，这个是我们在不用RxJava时，常用的开启网络请求的方法。它只是被包装在Observable中，最终还是会被调用来进行网络请求。



TODO:适配器or装饰模式

