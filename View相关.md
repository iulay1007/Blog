### View的部分继承关系

![IMG_0407(20210410-225759).PNG](https://i.loli.net/2021/04/10/hQRyUbH9trTvLSZ.jpg)





<!--more-->



### Android视图层次结构 : 

![a](https://i.loli.net/2020/05/03/z8kOwbsxcQF4Eo3.png)

每一个 **Activity **都持有一个 **Window** 对象，但是**Window**是一个**抽象类**，这里 Android 为 Window 提供了**唯一的实现类 **PhoneWindow。也就是说 Activity 中的 window 实例就是一个 PhoneWindow 对象。

但是 PhoneWindow 终究是 Window，它**并不具备多少 View 相关的能力**。

不过 PhoneWindow 中持有一个 Android 中非常重要的一个 View 对象 DecorView.

现在的关系就很明确了，**每一个 Activity 持有一个 PhoneWindow 的对象，而一个 PhoneWindow 对象持有一个 DecorView 的实例，所以 Activity 中 View 相关的操作其实大都是通过 DecorView 来完成。 **

DecorView是整个Window界面的**最顶层View**
一般情况下DecorView内部会包含一个竖直方向的LinearLayout，这个LinearLayout里面有上下两个部分，上面是**标题栏**，下面是**内容栏**。



### DecorView被加载到Window的过程

------



```java
 /**
     * Extended implementation of activity launch. Used when server requests a launch or relaunch.
     */
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }

        // Make sure we are running with the most recent config.
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);

        // Initialize before creating the activity
        if (!ThreadedRenderer.sRendererDisabled
                && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            HardwareRenderer.preload();
        }
        WindowManagerGlobal.initialize();

        // Hint the GraphicsEnvironment that an activity is launching on the process.
        GraphicsEnvironment.hintActivityLaunch();

        final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
                /*        handleResumeActivity(r.tolen, false, r.isForward, 
                !r.activity..mFinished && !r.startsNotResumed);
                */
            }
        } else {
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityTaskManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }

```



```java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //Activity创建
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    //调用onCreate（）方法
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            // updatePendingActivityConfiguration() reads from mActivities to update
            // ActivityClientRecord which runs in a different thread. Protect modifications to
            // mActivities to avoid race.
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```





Activity中的attach（）方法。包含了phoneWindow的创建。

```java
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
		//创建PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mAssistToken = assistToken;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);

        setAutofillOptions(application.getAutofillOptions());
        setContentCaptureOptions(application.getContentCaptureOptions());
    }
```



接下来看handleResumeActivity源码

```java
@Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // TODO Push resumeArgs into the activity for consideration
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        if (r == null) {
            // We didn't actually resume the activity, so skipping any follow-up actions.
            return;
        }
        if (mActivitiesToBeDestroyed.containsKey(token)) {
            // Although the activity is resumed, it is going to be destroyed. So the following
            // UI operations are unnecessary and also prevents exception because its token may
            // be gone that window manager cannot recognize it. All necessary cleanup actions
            // performed below will be done while handling destruction.
            return;
        }

        final Activity a = r.activity;

        if (localLOGV) {
            Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                    + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
        }

        final int forwardBit = isForward
                ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        // If the window hasn't yet been added to the window manager,
        // and this guy didn't finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
            try {
                willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
                        a.getActivityToken());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                
                //ViewRootImpl创建
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                  // WindowManager的实现类是WindowManagerImpl，
                // 所以实际调用的是WindowManagerImpl的addView方法
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig);
                if (DEBUG_CONFIGURATION) {
                    Slog.v(TAG, "Resuming activity " + r.activityInfo.name + " with newConfig "
                            + r.activity.mCurrentConfig);
                }
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }

            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        r.nextIdle = mNewActivities;
        mNewActivities = r;
        if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
        Looper.myQueue().addIdleHandler(new Idler());
    }

```





```

public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    ...

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mDisplay, mParentWindow);
    }
```



 WindowManagerGlobal的addView方法

```java
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View pannelParentView = null;
    synchronized (mLock) {
        ...
        // 创建ViewRootImpl实例
        root = new ViewRootImpl(view..getContext(), display);
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
    }
    try {
        // ViewRoot把DecorView加载到Window中
        root.setView(view, wparams, panelParentView);
    } catch (RuntimeException e) {
        synchronized (mLock) {
            final int index = findViewLocked(view, false);
            if (index >= 0) {
                removeViewLocked(index, true);
            }
        }
        throw e;
    }
}
```

ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的 

在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联 





setView中调用requestLayout()

```java
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
        //判断是否为主线 mLayoutRequested = true
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```



```java
 scheduleTraversals();
 void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
    
    
    //mTraversalRunnable
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    
    
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
    
    
  private void performTraversals() { 
  //主要方法
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); 
  performLayout(lp, mWidth, mHeight);
  performDraw();
  }
  
   private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

```

在View.class里

```java

public final void measure(int widthMeasureSpec, int heightMeasureSpec) { onMeasure(widthMeasureSpec, heightMeasureSpec);
}


protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```





### View绘制流程：

从ViewRoot的performTraversals方法开始，从上到下遍历整个视图树，每个View控件负责绘制自己，而ViewGroup还需要负责通知自己的子View进行绘制操作。

经过measure、layout、draw三个过程才能最终将一个View绘制出来。**measure**测量View的宽高，**layout**确定View在父容器中放置的位置，而draw则负责将Viewhui，而**draw**负责将View 绘制到屏幕上



performTraversals会依次调用performMeasure、performLayout、performDraw三个方法。这三个方法依次完成顶级View的measure、layout、draw。

其中，在performMeasure会调用measure方法，measure方法又会继续调用onMeasure方法。在onMeasure方法中则会对所有的子元素进行measure过程。这时候measure方法就会从父容器传递到子元素中。接着子元素重复类似的measure过程。如此反复就完成了整个View树的遍历。

performLayout、performDraw同理

![扫描全能王 2021-04-10 21.59.jpg](https://i.loli.net/2021/04/10/6WuGNKVUv4EajBl.jpg)

## measure：

 如果只是一个单一的View，只需要对自身进行measure就足够了，但如果是一个ViewGroup，除了对自身进行measure外还需要遍历它所有的子View(调用measureChildren()方法)，并调用其子view的measure，然后子view再执行以上递归操作。 



### LayoutParams：
LayoutParams携带了子控件针对父控件的信息，告诉父控件如何放置自己
LayoutParams类也只是简单的描述了宽高，宽和高都可以设置成三种值：
**1.一个确定的值** : 精确模式
**2.match_parent** ：精确模式
**3.wrap_content** ：最大模式，大小不定



### MeasureSpec：

作用：通过宽测量值widthMeasureSpec和高测量值heightMeasureSpec决定View的大小
组成：一个32位int值，高2位代表SpecMode(测量模式)，低30位代表SpecSize( 某种测量模式下的规格大小)。

**三种测量模式**：
**UNSPECIFIED**：父容器不对View有任何限制，要多大有多大。
**EXACTLY(精确模式)**：父视图为子视图指定一个确切的尺寸SpecSize。对应LayoutParams中的match_parent或具体数值。
**AT_MOST(最大模式)**：父容器为子视图指定一个最大尺寸SpecSize，View的大小不能大于这个值。对应LayoutParams中的wrap_content。不同view有不同的默认实现方式，比如TextView就默认包裹所有字符



**View的MeasureSpec是由父容器的MeasureSpec和自己的LayoutParams**决定的，但是对于DecorView来说有点不同，因为它没有父类。DecorView的MeasureSpec由**窗口的尺寸和自身的LayoutParams共同决定。**



![扫描全能王 2021-04-10 21.59_2.jpg](https://i.loli.net/2021/04/10/8DkvpY2zsKfjaQP.jpg)



ViewGroup的measure()和onMeasure()方法是从View继承过来的，没有做任何重写（measure方法是final限定，不能重写）。其onMeasure()方法由各个子类各自重写，实现自己的需求。ViewGroup的子类在onMeasure()中做的事其实都差不多:

**1.遍历子View**

**2.调用measureChild一类的方法让子View确定自己的大小**

**3.根据所有子View的大小确定自己的大小**

measureChild一类的方法内部会调用getChildMeasureSpec确定子View的测量模式，会调用child.measure(childWidthMeasureSpec, childHeightMeasureSpec)触发子View的测量。

**总结：测量始于DecorView，通过不断的遍历子View的measure方法，根据ViewGroup的MeasureSpec及子View的LayoutParams来决定子View的MeasureSpec，进一步获取子View的测量宽高，然后逐层返回，不断保存ViewGroup的测量宽高。**

## layout

在前面measure的作用是测量每个view的尺寸，而layout的作用是**根据前面测量的尺寸以及设置的其它属性值，共同来确定View的位置。**

单一view只需要进行layout就可以确定自己的四个顶点，而viewGroup除了layout过程外，还需要进行onLayout遍历所有的子view，并调用子view的layout方法，来确定子view的位置。

**layout方法主要的工作就是确定自身的四个顶点，然后调用setFrame方法确定view在父布局的位置**

onLayout方法的具体实现和具体的布局有关，所以View和View Group均没有具体的实现方法。

**所以ViewGroup除了上述过程之外，还需要重写onLayout方法来确定内部子view的位置**

总之，layout方法确定View本身的位置，而onLayout方法则会确定所有子元素的位置。





## draw

一般情况下View的draw流程分为四步：

**绘制背景 background.draw(canvas)
绘制自身内容（onDraw）
遍历子View，调用其draw方法把绘制过程分发下去（dispatchDraw）
绘制装饰（onDrawScrollBars）**



View绘制过程的传递是通过dispatchDraw来实现的，dispatchDraw方法会遍历调用所有子元素的draw方法。