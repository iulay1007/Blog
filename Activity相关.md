### Activity的基本状态

------

#### 活动状态(running)
活动状态一般是指该Activity正处于屏幕最显著的位置上显示，即该Activity是在Android活动栈的最顶端。 此时它处于可见并可和用户交互的状态。



#### **暂停状态(paused)**

暂停状态一般指该Activity已失去了焦点但仍然是可见的状态(包括部分可见)。不可以触摸操作。失去焦点即被一个新的非全屏的Activity或者一个透明的Activity覆盖



#### **停止状态(stopped)**

停止状态一般指该Activity被另一个Activity完全覆盖的状态

此状态下，该Activity的数据会暂时保留，但是，一旦系统需要内存，这种处于Stopped状态的Activity占用的空间会优先被清理并重新利用



#### 销毁状态(killed)

此时Activity已从Activity栈中移除，在内存中不存在



### Activity的状态转换

------

![屏幕截图(65).png](https://i.loli.net/2020/11/27/sfaq5YNBzuGKWvb.png)



### Activity的生命周期

------



![屏幕截图(63).png](https://i.loli.net/2020/11/27/bUiop372zD6X8gx.png)



***onCreate ( )***: 首次创建Activity时调用。进行Activity的一些初始化工作，比如使用setContentView加载布局，对一些控件和变量进行初始化等。



***onStart ( )*** : 此时Activity已经**可见**了，但是还在后台，我们还看不到，无法与Activity交互。可以理解为Activity已经显示出来了，但是我们还看不见



***onResume ( )*** : Activity在这个阶段已经出现在前台并且**可见**了（处于running状态）。



***onPause ( )*** :表示暂停(paused状态），当Activity要跳到另一个Activity或应用正常退出时都会执行这个方法。我们可以进行一些轻量级的存储数据和去初始化的工作，不能太耗时，因为在跳转Activity时只有当一个Activity执行完了onPause方法后另一个Activity才会启动，而且Android中指定如果onPause在500ms即0.5秒内没有执行完毕的话就会强制关闭Activity。



***onStop ( )*** :表示即将停止，此时Activity不可见。当Activity即将被销毁或被另一个Activity覆盖，则会调用onStop。如果新的Activity采用了透明主题，则不会回调onStop。这个阶段可以做一些微重量级的回收工作，同样不能太耗时。



***onDestroy ( )*** : Activity被销毁时调用。点击返回键或Activity里调用了finish方法或系统内存不足而选择强杀app时会调用onDestroy。在这里可以做一些回收工作和最终的资源释放。



***onRestart( )*** ：表示重新开始，Activity在这时**可见**，当用户按Home键切换到桌面后又切回来或者从后一个Activity切回前一个Activity就会触发这个方法。



#### 

从整个生命周期来说，***onCreate和onDestroy***是配对的，分别标识着Activity的创建和销毁，并且只会调用一次。

从Activity是否在前台来说，***onResume和onPause***是配对的，随着用户的操作或设备屏幕的点亮和熄灭，这两个方法可以被多次调用。

从Activity是否可见来说，***onStart和onStop***是配对的，随着用户的操作或设备屏幕的点亮和熄灭，这两个方法可以被多次调用。





#### 打开一个app的MainActivity并跳转到另一个Activity时的生命周期

```
//某一个Activity第一次启动，回调为：onCreate->onStart->onResume
/com.example.activitytest I/.MainActivity :onCreate
/com.example.activitytest I/.MainActivity :onStart
/com.example.activitytest I/.MainActivity :onResume

//跳转到下一个Activity时
当需要从MainActivity切换到SecondActivity时，先执行MainActivity中的与onResume()相对应的onPause()操作，比如关闭独占设备(比如相机），或其它耗费cpu的操作；
以防SecondActivity也需要使用这些资源，关闭耗CPU的操作，也有利于SecondActivity运行的流畅。

/com.example.activitytest I/.MainActivity :onPause
/com.example.activitytest I/.SecondActivity :onCreate
/com.example.activitytest I/.SecondActivity :onStart
/com.example.activitytest I/.SecondActivity :onResume
/com.example.activitytest I/.MainActivity :onStop


```



#### 为什么不先执行MainActivity的onStop再执行SecondActivity的onCreate?

从用户体验的角度来分析,当用户触发某事件切换到新的Activity，用户肯定是想尽快进入新的视图进行操作。
MainActivity中比较消耗资源的部分在onPause时关闭后，再切换到SecondActivity中执行SecondActivity
的初始化，显示SecondActivity中的View后，用户可以进行交互。

此时后台再去执行MainActivity的onStop()操作，即使这里面有些比较耗时的操作，也没有关系，这是在后台执行所以也不影响用户的体验。





### Activity的优先级

------

从高到低：

(1)前台Activity——正在与用户交互的Activity，优先级最高

(2)可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但位于后台无法和用户直接交互

(3)后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低

当系统内存不足时，会按照上述优先级去杀死Activity所在进程。





### Activity的启动模式

------

#### standard

**标准模式**：如果不设置Activity的启动模式，系统会默认将其设置为standard。

这种模式下，同一个Activity可以有多个实例，每次启动Activity，无论任务栈中是否已经有这个Activity的实例，系统都会创建一个新的Activity实例。

![20180606100919308.png](https://i.loli.net/2020/11/29/bnKkBhFUXQwjSmY.png)



#### singleTop

**栈顶复用模式**：在这种模式下，如果新启动的Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建。

如果新Activity的实例已存在但不位于栈顶，那么新Activity仍会重新创建。

![2.png](https://i.loli.net/2020/11/29/lUGdxmt6nOkog5X.png)



#### singleTask

**栈内复用模式**：这是一种单实例模式，一个栈中同一个Activity只存在唯一一个实例，无论是否在栈顶，只要存在实例，都不会重新创建。但Activity已经存在但不位于栈顶时，系统就会把该Activity移到栈顶。会导致任务栈内它上面的Activity被销毁。

![3.png](https://i.loli.net/2020/11/29/mOZjVPEzFMqaH3k.png)



#### singleInstance

**单实例模式**：这是一种加强的singleTask模式，它除了具有singleTask模式的所有特性外，还加强了一点，那就是此种模式的Activity只能单独地位于一个任务栈中，不同的应用去打开这个Activity 共用同一个Activity。在该模式下，我们会为目标Activity创建一个新的任务栈，将目标Activity放入新的任务栈，由于栈内复用的特性，后续的请求不会创建新的Activity，除非这个独特的任务栈被系统销毁了。

![4.png](https://i.loli.net/2020/11/29/uxdQ1EsDAoCOfIt.png)