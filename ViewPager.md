# ViewPager及懒加载与预加载分析

## ViewPager的使用

### 常见用法

网上一些教程上会运用以下方式在PagerAdapter中获取Fragment实例，即在构造PagerAdapter时传入相应的Fragment列表，也就是在构造PagerAdapter前已经将所有Fragment进行了初始化。

```java
public class TestFragmentPagerAdapter extends FragmentPagerAdapter {

    private List<Fragment> fragmentList;
    public TestFragmentPagerAdapter(FragmentManager fm, List<Fragment> fragmentList){
        super(fm);
        this.fragmentList = fragmentList;
    }

    @NonNull
    @Override
    public Fragment getItem(int position) {
        return fragmentList.get(position);
    }

    @Override
    public int getCount() {
        return fragmentList.size();
    }
}
```

### 官方用法

但查看官方示例代码可发现，官方更加倾向于以下创建Fragment实例的方式。即在getItem()中开始创建实例，而非将所有实例创建好然后传入PagerAdapter

```java
public static class MyAdapter extends FragmentPagerAdapter {
        public MyAdapter(FragmentManager fm) {
            super(fm, BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT);
        }

        @Override
        public int getCount() {
            return NUM_ITEMS;
        }

        @Override
        public Fragment getItem(int position) {
            return ArrayListFragment.newInstance(position);
        }
    }

    /*
     *ListFragment is the same as a ListActivity - it's just a Fragment,
     *that extends List methods. You can just add Fragment, that contains ListView and implement
     *all required for list methods, like in Activity
     */
    public static class ArrayListFragment extends ListFragment {
        int mNum;

        /**
         * Create a new instance of CountingFragment, providing "num"
         * as an argument.
         */
        static ArrayListFragment newInstance(int num) {
            ArrayListFragment f = new ArrayListFragment();

            // Supply num input as an argument.
            /*
            为什么不通过构造函数直接传参数？详见
https://blog.csdn.net/tu_bingbing/article/details/24143249
			*/
            Bundle args = new Bundle();
            args.putInt("num", num);
            f.setArguments(args);

            return f;
        }

        /**
         * When creating, retrieve this instance's number from its arguments.
         */
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            mNum = getArguments() != null ? getArguments().getInt("num") : 1;
        }

        /**
         * The Fragment's UI is just a simple text view showing its
         * instance number.
         */
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                                 Bundle savedInstanceState) {
            View v = inflater.inflate(R.layout.fragment_pager_list, container, false);
            View tv = v.findViewById(R.id.text);
            ((TextView) tv).setText("Fragment #" + mNum);
            return v;
        }

        @Override
        public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
            super.onViewCreated(view, savedInstanceState);
            setListAdapter(new ArrayAdapter<String>(getActivity(),
                    android.R.layout.simple_list_item_1));
        }

        @Override
        public void onListItemClick(ListView l, View v, int position, long id) {
            Log.i("FragmentList", "Item clicked: " + id);
        }
    }
```



> getItem()方法中创建实例也可以使用工厂模式



为什么不推荐提前初始化所有Fragment？

虽然大部分Fragment的初始化并未有太大的消耗，但不排除一些特殊情况，比如在Fragment中进行例如new一个较大的数组这种操作。



## ViewPager及PagerAdapter实现类分析

### ViewPager源码分析

从ViewPager的setAdapter()方法看起。

```java
public void setAdapter(@Nullable PagerAdapter adapter) {
    //如果存在旧的Adapter,销毁旧的Adapter的数据
        if (mAdapter != null) {
            mAdapter.setViewPagerObserver(null);
            mAdapter.startUpdate(this);
            //销毁旧Item的数据
            for (int i = 0; i < mItems.size(); i++) {
                final ItemInfo ii = mItems.get(i);
                mAdapter.destroyItem(this, ii.position, ii.object);
            }
            mAdapter.finishUpdate(this);
            mItems.clear();
            removeNonDecorViews();
            mCurItem = 0;
            scrollTo(0, 0);
        }

        final PagerAdapter oldAdapter = mAdapter;
        mAdapter = adapter;
        mExpectedAdapterCount = 0;

        if (mAdapter != null) {
            if (mObserver == null) {
                mObserver = new PagerObserver();
            }
            mAdapter.setViewPagerObserver(mObserver);
            mPopulatePending = false;
            final boolean wasFirstLayout = mFirstLayout;
            mFirstLayout = true;
            mExpectedAdapterCount = mAdapter.getCount();
            if (mRestoredCurItem >= 0) {
                mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
                setCurrentItemInternal(mRestoredCurItem, false, true);
                mRestoredCurItem = -1;
                mRestoredAdapterState = null;
                mRestoredClassLoader = null;
            } else if (!wasFirstLayout) {
                populate();
            } else {
                //onMeasure()中也会调用populate()方法
                requestLayout();
            }
        }

        // Dispatch the change to any listeners
        if (mAdapterChangeListeners != null && !mAdapterChangeListeners.isEmpty()) {
            for (int i = 0, count = mAdapterChangeListeners.size(); i < count; i++) {
                mAdapterChangeListeners.get(i).onAdapterChanged(this, oldAdapter, adapter);
            }
        }
    }
```

正常情况下最终会调用populate()方法。

populate()方法主要代码如下：

```java
    void populate(int newCurrentItem) {
    //adapter页面将更改时调用
    mAdapter.startUpdate(this);
        
    final int N = mAdapter.getCount();
        
    // Locate the currently focused item or add it if needed.
    int curIndex = -1;
    ItemInfo curItem = null;
    for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
            final ItemInfo ii = mItems.get(curIndex);
            if (ii.position >= mCurItem) {
                //查到时
                if (ii.position == mCurItem) curItem = ii;
               //查找到或者往后不会再找到
                break;
            }
        }
    //找不到当前需要的item,则add    
    if (curItem == null && N > 0) {
        curItem = addNewItem(mCurItem, curIndex);
        }
        
    //...两个循环对左右两边的Fragment进行缓存及清除(代码略)
        
    mAdapter.setPrimaryItem(this, mCurItem, curItem.object);
   
}

	static class ItemInfo {
        Object object;
        int position;
        boolean scrolling;
        float widthFactor;
        float offset;
    }
	//存放缓存的Fragment
	private final ArrayList<ItemInfo> mItems = new ArrayList<ItemInfo>();
    int mCurItem;   // Index of currently displayed page.


```

查看addNewItem对应的方法

```java
 ItemInfo addNewItem(int position, int index) {
        ItemInfo ii = new ItemInfo();
        ii.position = position;
        ii.object = mAdapter.instantiateItem(this, position);
        ii.widthFactor = mAdapter.getPageWidth(position);
        if (index < 0 || index >= mItems.size()) {
            mItems.add(ii);
        } else {
            mItems.add(index, ii);
        }
        return ii;
    }
```

可以看到这里调用了mAdapter.instantiateItem()方法，它的实现类是setAdapter()设置的PagerAdapter。接下来看一下PagerAdapter的两个实现类。



### 直观感受两种PagerAdapter实现类的区别

------

#### FragmentPagerAdapter

- 适用于在固定的少量同级屏幕之间进行导航。

- 默认预加载当前Fragment及左右两个Fragment，若左边或右边无Fragment则不加载。

  ![IMG_0559.PNG](https://i.loli.net/2021/10/15/BRjuCawQDktniX7.jpg)

- > 当前Fragment即为ViewPager中的CurrentItem,默认设置为0
  >
  > 默认加载的左右两个Fragment，是根据mOffscreenPageLimit计算出的，mOffscreenPageLimit默认为1，所以默认预加载左右各一个Fragment
  >
  > 本例均使用默认值

  在加载最左边，即第一个Fragment时，生命周期如下。

  可见当前Fragment及其右边的Fragment都会执行到onResume()方法


![屏幕截图(229).png](https://i.loli.net/2021/10/11/4UV6ClNTWhGDnYr.png)



- 当继续切换到第三个Fragment时，默认会缓存第二个Fragment和第四个Fragment，而刚开始已经加载过的Fragment生命周期会执行到onDestroyView(),并未执行onDestroy()方法，即Fragment的实例数据还存在，只是View被销毁了

  ![IMG_0560.PNG](https://i.loli.net/2021/10/15/WBaQDzhiG9oLNpm.jpg)


#### FragmentStatePagerAdapter

- 适用于对未知数量的页面进行分页。`FragmentStatePagerAdapter` 会在用户导航至其他位置时销毁 Fragment，从而优化内存使用情况
- 默认刚开始加载的Fragment的生命周期同上。

- 当切换到第三个Fragment时，最左边的Fragment会执行到onDetach()方法，即被销毁了。图略。




### 两种PagerAdapter实现类源码分析

------

#### instantiateItem()方法

##### FragmentPagerAdapter

```java
public Object instantiateItem(@NonNull ViewGroup container, int position) {
        if (this.mCurTransaction == null) {
            this.mCurTransaction = this.mFragmentManager.beginTransaction();
        }

    	//根据position得到itemId,该方法默认返回position
        long itemId = this.getItemId(position);
    
    	//生成tag,默认返回 "android:switcher:" + viewId + ":" + id
        String name = makeFragmentName(container.getId(), itemId);
    	
    	//根据tag到FragmentManager中寻找Fragment的实例
        Fragment fragment = this.mFragmentManager.findFragmentByTag(name);
    	
    	//找到就调用attach方法
        if (fragment != null) {
            this.mCurTransaction.attach(fragment);
        } else {
        //否则调用getItem()获取Fragment实例，该方法是创建该FragmentPagerAdapter时会重			//写的
            fragment = this.getItem(position);
            this.mCurTransaction.add(container.getId(), fragment, makeFragmentName(container.getId(), itemId));
        }
		//判断当前Fragment是否为获得焦点的当前Fragment
        if (fragment != this.mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            if (this.mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                this.mCurTransaction.setMaxLifecycle(fragment, State.STARTED);//1
            } else {
                fragment.setUserVisibleHint(false);
            }
        }

        return fragment;
    }
public long getItemId(int position) {
        return (long)position;
    }

    private static String makeFragmentName(int viewId, long id) {
        return "android:switcher:" + viewId + ":" + id;
    }
	public static final int BEHAVIOR_SET_USER_VISIBLE_HINT = 0;
    public static final int BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT = 1;
```

> - 如果 behavior 的值为 `BEHAVIOR_SET_USER_VISIBLE_HINT`，那么当 Fragment 对用户的可见状态发生改变时，`setUserVisibleHint` 方法会被调用。
> - 如果 behavior 的值为 `BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT` ，那么当前选中的 Fragment 在 `Lifecycle.State#RESUMED` 状态 ，其他不可见的 Fragment 会被限制在 `Lifecycle.State#STARTED` 状态。
>
> mBehavior的值默认为BEHAVIOR_SET_USER_VISIBLE_HINT，如果想要设置，可以在自定义的FragmentPagerAdapter的构造函数中调用super(fm,FragmentPagerAdapter.BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT);

注释1处，上述例子如果设置了mBehavior为BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT，则该预加载的Fragment生命周期只会执行到onStart()，原因是调用了setMaxLifecycle()。该方法可以设置活跃状态下 Fragment 最大的状态，如果该 Fragment 超过了设置的最大状态，那么会强制将 Fragment 降级到相应的状态。[详细可以看这篇博客](https://juejin.cn/post/6844904033774206984)



##### FragmentStatePagerAdapter

```java
//SavedState是Fragment类里的，用于保存Bundle的信息
private ArrayList<SavedState> mSavedState;
//并非保存所有Fragment，而是保存根据mOffscreenPageLimit算出来的个数的Fragment
private ArrayList<Fragment> mFragments;

public Object instantiateItem(@NonNull ViewGroup container, int position) {
        Fragment fragment;
    	//先判断是否缓存了该Fragment，若有则直接复用
        if (this.mFragments.size() > position) {
            fragment = (Fragment)this.mFragments.get(position);
            if (fragment != null) {
                return fragment;
            }
        }

        if (this.mCurTransaction == null) {
            this.mCurTransaction = this.mFragmentManager.beginTransaction();
        }
    
		//getItem是自行重写的方法
        fragment = this.getItem(position);
    
    	//查询是否保存有对应的Bundle信息
        if (this.mSavedState.size() > position) {
            SavedState fss = (SavedState)this.mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }

        while(this.mFragments.size() <= position) {
            this.mFragments.add((Object)null);
        }
		
        fragment.setMenuVisibility(false);
        if (this.mBehavior == BEHAVIOR_SET_USER_VISIBLE_HINT) {
            fragment.setUserVisibleHint(false);
        }
		
    	//保存Fragment信息
        this.mFragments.set(position, fragment);
        this.mCurTransaction.add(container.getId(), fragment);
        if (this.mBehavior ==  BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
            this.mCurTransaction.setMaxLifecycle(fragment, State.STARTED);
        }

        return fragment;
    }
```



#### destroyItem()方法

##### FragmentPagerAdapter

```java
 public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        Fragment fragment = (Fragment)object;
        if (this.mCurTransaction == null) {
            this.mCurTransaction = this.mFragmentManager.beginTransaction();
        }
		
     	/*
     	detach是从UI中将fragment的元素去掉，但是依然保留状态，
     	当调用attach的时候重新将之前的fragment连同状态一起恢复。 
     	*/
        this.mCurTransaction.detach(fragment);
        if (fragment == this.mCurrentPrimaryItem) {
            this.mCurrentPrimaryItem = null;
        }

    }
```

##### FragmentStatePagerAdapter

```Java
 public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        Fragment fragment = (Fragment)object;
        if (this.mCurTransaction == null) {
            this.mCurTransaction = this.mFragmentManager.beginTransaction();
        }

        while(this.mSavedState.size() <= position) {
            this.mSavedState.add((Object)null);
        }

        this.mSavedState.set(position, fragment.isAdded() ? this.mFragmentManager.saveFragmentInstanceState(fragment) : null);
        this.mFragments.set(position, (Object)null);

     	//remove是将fragment从UI中去掉，状态无法恢复了
        this.mCurTransaction.remove(fragment);
        if (fragment == this.mCurrentPrimaryItem) {
            this.mCurrentPrimaryItem = null;
        }

    }
```

通过两个destroyItem()的实现可以发现，对fragment的不同销毁方法决定了fragment的生命周期。这也是为什么之前讲的，两种PagerAdapter对于超出预加载范围的Fragment的销毁，FragmentPagerAdapter对应的Fragment会执行到onDestroyView()方法，而FragmentStatePagerAdapter对应的Fragment会执行到onDetach()方法。

#### setPrimaryItem()方法

以上两个PagerAdapter的子类对其的实现是一样的

```java
public void setPrimaryItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
    	//fragment是将要显示的Fragment，mCurrentPrimaryItem是目前正在显示的Fragment
        Fragment fragment = (Fragment)object;
        if (fragment != this.mCurrentPrimaryItem) {
            if (this.mCurrentPrimaryItem != null) {
                this.mCurrentPrimaryItem.setMenuVisibility(false);
                if (this.mBehavior == 1) {
                    if (this.mCurTransaction == null) {
                        this.mCurTransaction = this.mFragmentManager.beginTransaction();
                    }
		//将当前显示的Fragment的MaxLifecycle设置为STARTED。
        //即其生命周期从onResume回退到onStart
                    this.mCurTransaction.setMaxLifecycle(this.mCurrentPrimaryItem, State.STARTED);
                } else {
                    this.mCurrentPrimaryItem.setUserVisibleHint(false);
                }
            }

            fragment.setMenuVisibility(true);
            if (this.mBehavior == 1) {
                if (this.mCurTransaction == null) {
                    this.mCurTransaction = this.mFragmentManager.beginTransaction();
                }
			//将即将显示的Fragment的MaxLifecycle设置为RESUMED
            //所以该fragment生命周期从onStart变成onResume
                this.mCurTransaction.setMaxLifecycle(fragment, State.RESUMED);
            } else {
                fragment.setUserVisibleHint(true);
            }

            this.mCurrentPrimaryItem = fragment;
        }

    }
```



## 预加载和懒加载

### 预加载

#### 概念

  ViewPager的预加载机制会默认一次加载当前页面前后两个页面，预加载界面的个数可以自行设置。但至少会预加载前后各一个界面。

#### 优缺点

预加载在一定程度上可以保证切换界面时可以很快地见到加载完数据的完整界面，但其代价是，在不切换到别的界面时，别的界面会进行耗时的获取数据操作，这可能导致当前界面获取数据时发生卡顿。

### 懒加载

#### 概念

懒加载也可以叫做延迟加载，即当界面可见时再进行数据的加载。

#### 作用

懒加载可以通过对数据的延迟加载，解决因为预加载导致的性能降低的问题。

#### 实现

##### 传统处理方式

> - 使用 ViewPager切换 Fragment 页面时（已经初始化完毕），只有 setUserVisibleHint(boolean isVisibleToUser) 会被回调。
>
> - setUserVisibleHint(boolean isVisibleToUser) 方法总是会优先于 Fragment 生命周期函数的调用。

```java
public abstract class BaseLazyFragment extends Fragment {
	//当前Fragment是否可见
    private boolean isVisibleToUser = false;
    
	//Fragment的View是否被创建
    private boolean isViewCreated = false;

    //是否第一次加载数据
    private boolean isFirstLoad = true;

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        this.isVisibleToUser = isVisibleToUser;
        //当setUserVisibleHint被回调时，会调用懒加载方法，这里其实可以进一步判断处理
        //如当可见时进行加载操作，不可见时停止加载
        onLazyLoad();
    }


    @Override
    public void onViewCreated(@NonNull @NotNull View view, @Nullable @org.jetbrains.annotations.Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        //View创建成功
        isViewCreated = true;
        //这个函数被真正调用主要在于第一个Fragment加载的时候
        //因为最开始setUserVisibleHint执行时，view还未创建。
        onLazyLoad();
    }
    
        private void onLazyLoad() {
        if(isVisibleToUser && isViewCreated && isFirstLoad){
            isFirstLoad = false;
            lazyLoad();
        }
    }
	//子类实现具体的加载逻辑
    protected abstract void lazyLoad();
}

```

##### Androidx 下的懒加载

Google 在 Androidx 在 FragmentTransaction 中增加了 setMaxLifecycle 方法来控制 Fragment 所能调用的最大的生命周期函数。

该方法可以设置活跃状态下 Fragment 最大的状态，如果该 Fragment 生命周期超过了设置的最大状态，那么会强制将 Fragment 降级到设置的状态。

结合之前讲的，可以在PagerAdapter子类的构造函数中调用super(fm,BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT);来实现当前Fragment生命周期执行到onResume,预加载的Fragment生命周期执行到onStart,这样，就不会存在提前加载数据的情况了。

因此懒加载的方式变得更加简洁，不再需要多个标志位判断这样不优雅的代码实现懒加载的功能了

```kotlin
abstract class LazyLifecycleFragment : Fragment() {
    private var isFirstLoad = true

    override fun onResume() {
        super.onResume()
        if (isFirstLoad) {
        	isFirstLoad = false
            loadData()
        }
    }

    abstract fun loadData()
}
```



预加载和懒加载都有一定的应用场景，在数据加载量少的时候不用懒加载也不会有很大的影响。

而懒加载的具体代码也可以根据具体的业务逻辑来确定，比如加载过数据的Fragment被destroyItem()方法处理后，再显示该Fragment的逻辑，因为从前面的分析可以知道具体的destroyItem()逻辑在不同PagerAdapter的实现类中是不一样的。

所以上面的代码只是最基础的代码逻辑参考，具体的逻辑可以在此基础上修改。



## ViewPager2

### 简介

ViewPager2 是 ViewPager 库的改进版本，可提供增强型功能并解决使用 ViewPager时遇到的一些常见问题。

### 特性

- 垂直方向支持

- DiffUtil

  > ViewPager2在 RecyclerView的基础上构建而成，这意味着它可以访问 DiffUtil 实用程序类。这一点带来了多项优势，但最突出的一项是，这意味着 ViewPager2 对象本身会利用 RecyclerView 类中的数据集更改动画

- 可修改的 Fragment 集合

  > ViewPager2 支持对可修改的 Fragment 集合进行分页浏览，在底层集合发生更改时调用 notifyDatasetChanged() 来更新界面。
  >
  > 这意味着，您的应用可以在运行时动态修改 Fragment 集合，而 ViewPager2 会正确显示修改后的集合。


### ViewPager迁移到 ViewPager2

#### 主要代码变化

1. 将父类更改为 RecyclerView.Adapter 用于分页浏览视图，或更改为 FragmentStateAdapter 用于分页浏览 Fragment。

2. 更改基于 Fragment 的适配器类中的构造函数参数。

3. 替换 `getItemCount()`，而不是 `getCount()`。

4. 在基于 Fragment 的适配器类中，替换 createFragment()，而不是 getItem()。


#### 若使用了TabLayout

 TabLayout 与 ViewPager 集成使用的是它自己的 setupWithViewPager() 方法，但是如果要与 ViewPager2 集成，却需要使用 TabLayoutMediator 实例。

TabLayoutMediator 对象还可处理为 TabLayout 对象生成页面标题的任务，这意味着相关适配器类不需要替换 getPageTitle()

示例如下：

```kotlin
            val tabLayout = view.findViewById(R.id.tab_layout)
            TabLayoutMediator(tabLayout, viewPager) { tab, position ->
                tab.text = "OBJECT ${(position + 1)}"
            }.attach()
```



### OffscreenPageLimit

查看相应的setOffscreenPageLimit()方法

```java
public static final int OFFSCREEN_PAGE_LIMIT_DEFAULT = -1;

private @OffscreenPageLimit int mOffscreenPageLimit = OFFSCREEN_PAGE_LIMIT_DEFAULT;

public void setOffscreenPageLimit(@OffscreenPageLimit int limit) {
        if (limit < 1 && limit != OFFSCREEN_PAGE_LIMIT_DEFAULT) {
            throw new IllegalArgumentException(
                    "Offscreen page limit must be OFFSCREEN_PAGE_LIMIT_DEFAULT or a number > 0");
        }
        mOffscreenPageLimit = limit;
        // Trigger layout so prefetch happens through getExtraLayoutSize()
        mRecyclerView.requestLayout();
    }
```

ViewPager2的limit必须大于0或者为-1。limit的值不应该过大，尤其是在界面复杂的情况下。

limit的值默认设置为OFFSCREEN_PAGE_LIMIT_DEFAULT（即-1）。OFFSCREEN_PAGE_LIMIT_DEFAULT表示使用RecyclerView的默认缓存机制(刚开始只加载当前界面)，而不是显式地将页面预取并保留到当前页面的任一侧。



当加载第一个界面时，其对应加载的Fragment的生命周期如下![屏幕截图(231).png](https://i.loli.net/2021/10/13/LA7KSwpDPCihGMb.png)

当切换到下一个Fragment时，生命周期如下:

第一个Fragment的生命周期会到onPause()

![屏幕截图(233).png](https://i.loli.net/2021/10/15/ezgFpdrPhQXcBis.png)

继续切换到下一个Fragment，生命周期如下。

可以发现，此时并不会对第一个Fragment进行销毁。

![屏幕截图(235).png](https://i.loli.net/2021/10/15/2cfkJgrST4ma1uB.png)



> [关于RecyclerView的默认缓存机制和预取机制及其对Fragment的销毁情况可以看这个博客](https://www.jianshu.com/p/1d95e729c571)



### 懒加载

当通过setOffscreenPageLimit()设置了limit不为-1时，预加载的Fragment生命周期只执行到onStart()。显然是在FragmentStateAdapter中设置了MaxLifecycle，此时的懒加载方案和ViewPager在Androidx下的懒加载方案一致。

而当OffscreenPageLimit值为-1时，不存在预加载的情况，所以就不需要进行懒加载处理