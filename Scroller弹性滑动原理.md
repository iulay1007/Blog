### Scroller弹性滑动原理

```java
  Scroller mScroller = new Scroller(context);
  
//供外部调用的自定义的方法
  public void smoothScrollTo(int destX,int destY){
        int scrollX=getScrollX();
        int delta=destX-scrollX;
        mScroller.startScroll(scrollX,0,delta,0,2000);
        invalidate();
    }
 
  
```

```java
 //smoothScrollTo调用了startScroll方法，源码如下
 public void startScroll(int startX, int startY, int dx, int dy, int duration) {
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAnimationTimeMillis();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float) mDuration;
    }
```

startScroll（）方法并没有调用类似开启滑动的方法，而是保存了传进来的各种参数。startX和startY表示滑动开始的起点，dx和dy表示滑动的距离，duration表示滑动持续的时间。所以startScroll（）方法只是用来做前期的准备，并不能使View进行滑动。

关键是我们在startScroll（）方法后调用了 invalidate（）方法，这个方法会导致View的重绘，而View的重绘会调用View的draw（）方法，draw（）方法又会调用View的computeScroll（）方法。我们可以重写该方法实现想要的功能

```java

 @Override
    public void computeScroll() {
        if(mScroller.computeScrollOffset()){
            scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
            postInvalidate();
        }
    }
```

```java
//computeScroll()方法中的computeScrollOffset()方法
public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    
        if (timePassed < mDuration) {
            switch (mMode) {
            case SCROLL_MODE:
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                final float t = (float) timePassed / mDuration;
                final int index = (int) (NB_SAMPLES * t);
                float distanceCoef = 1.f;
                float velocityCoef = 0.f;
                if (index < NB_SAMPLES) {
                    final float t_inf = (float) index / NB_SAMPLES;
                    final float t_sup = (float) (index + 1) / NB_SAMPLES;
                    final float d_inf = SPLINE_POSITION[index];
                    final float d_sup = SPLINE_POSITION[index + 1];
                    velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                    distanceCoef = d_inf + (t - t_inf) * velocityCoef;
                }

                mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
                
                mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
                // Pin to mMinX <= mCurrX <= mMaxX
                mCurrX = Math.min(mCurrX, mMaxX);
                mCurrX = Math.max(mCurrX, mMinX);
                
                mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
                // Pin to mMinY <= mCurrY <= mMaxY
                mCurrY = Math.min(mCurrY, mMaxY);
                mCurrY = Math.max(mCurrY, mMinY);

                if (mCurrX == mFinalX && mCurrY == mFinalY) {
                    mFinished = true;
                }

                break;
            }
        }
        else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
    
```

首先会计算动画持续时间timePassed。如果动画持续时间小于我们设置的滑动持续时间mDuration，则执行switch语句。因为在startScroll（）方法中的mMode值为SCROLL_MODE，则执行该分支语句，然后根据插值器Interpolator来计算出在该时间段内移动的距离，赋值给mCurrX和mCurrY。这样我们就能头发Scroller来获取当前的ScrollX和ScrollY了。另外，computeScrollOffset（）的返回值如果为true则表示滑动未结束，为false表示滑动结束。如果滑动未结束，我们就持续调用scrollTo（）方法和invalida（）方法来进行View的滑动。



总结Scroller的原理：Scroller并不能直接实现View的滑动，它需要配合View的computeScroll（）方法。在computeScroll（）中不断让View进行重绘，每次重绘都会计算滑动持续的时间，根据这个持续时间就能算出这次View的滑动位置。我们根据每次的滑动位置调用scrollTo方法（）进行滑动。这样不断地重复上述过程就形成了弹性滑动。