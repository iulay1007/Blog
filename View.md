### Android视图层次结构 : 

![a](https://i.loli.net/2020/05/03/z8kOwbsxcQF4Eo3.png)

每一个 **Activity **都持有一个 **Window** 对象，但是**Window**是一个**抽象类**，这里 Android 为 Window 提供了**唯一的实现类 **PhoneWindow。也就是说 Activity 中的 window 实例就是一个 PhoneWindow 对象。

但是 PhoneWindow 终究是 Window，它**并不具备多少 View 相关的能力**。

不过 PhoneWindow 中持有一个 Android 中非常重要的一个 View 对象 DecorView.

现在的关系就很明确了，**每一个 Activity 持有一个 PhoneWindow 的对象，而一个 PhoneWindow 对象持有一个 DecorView 的实例，所以 Activity 中 View 相关的操作其实大都是通过 DecorView 来完成。 **

DecorView是整个Window界面的**最顶层View**
一般情况下DecorView内部会包含一个竖直方向的LinearLayout，这个LinearLayout里面有上下两个部分，上面是**标题栏**，下面是**内容栏**。



### View工作流程：measure测量->layout布局->draw绘制

**measure**确定View的测量宽高
**layout**确定View的最终宽高和四个顶点的位置
**draw**将View 绘制到屏幕上

## measure：

 如果只是一个单一的view，只需要对自身进行measure就足够了，但如果是一个viewgroup，除了对自身进行measure外还需要遍历它所有的子view，并调用其子view的measure，然后子view再执行以上递归操作。 



### LayoutParams：
LayoutParams携带了子控件针对父控件的信息，告诉父控件如何放置自己
LayoutParams类也只是简单的描述了宽高，宽和高都可以设置成三种值：
**1.一个确定的值**
**2.match_parent**
**3.wrap_content**



### MeasureSpec：

作用：通过宽测量值widthMeasureSpec和高测量值heightMeasureSpec决定View的大小
组成：一个32位int值，高2位代表SpecMode(测量模式)，低30位代表SpecSize( 某种测量模式下的规格大小)。
三种模式：
**UNSPECIFIED**：父容器不对View有任何限制，要多大有多大。
**EXACTLY(精确模式)**：父视图为子视图指定一个确切的尺寸SpecSize。对应LayoutParams中的match_parent或具体数值。
**AT_MOST(最大模式)**：父容器为子视图指定一个最大尺寸SpecSize，View的大小不能大于这个值。对应LayoutParams中的wrap_content。不同view有不同的默认实现方式，比如TextView就默认包裹所有字符



**View的MeasureSpec是由父容器的MeasureSpec和自己的LayoutParams**决定的，但是对于DecorView来说有点不同，因为它没有父类。DecorView的MeasureSpec由**窗口的尺寸和自身的LayoutParams共同决定。**

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

**viewGroup除了上述过程之外，还需要重写onLayout方法来确定内部子view的位置**

总之，layout方法确定View本身的位置，而onLayout方法则会确定所有子元素的位置。





## draw

一般情况下View的draw流程分为四步：

**绘制背景（drawBackground）
绘制自身内容（onDraw）
遍历子View，调用其draw方法把绘制过程分发下去（dispatchDraw）
绘制装饰（onDrawForeground）**