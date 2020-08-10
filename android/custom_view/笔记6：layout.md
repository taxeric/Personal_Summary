经历了`measure`后，系统确定了View的大小，接下来就进入到Layout的过程。**layout用于确定子View在ViewGroup中的位置**。与measure一致，该过程也有一个`onLayout`方法，且`layout`也是起到一个调度的作用，但与measure不同的是，layout方法会保存传进来的尺寸和位置（测量阶段是父View统一保存所有子View的布局，而布局阶段是子View自个保存自己的布局）。

看layout的源码
```java
    public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        ...
    }
```
可以看到在onLayout方法被调用之前，会做一个判断：isLayoutModeOptical，然后分别调用方法，而点击`setOpticalFrame`方法能看到最终调用也是`setFrame`，所以看看这个方法做了啥
```java
        /**
     * Assign a size and position to this view.
     * 为该视图分配尺寸和位置
     * This is called from layout.
     * 应该被调用在layout中 
     */
    protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;
        ...
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            ...

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            ...

            mLeft = left;
            mTop = top;
            mRight = right;
            mBottom = bottom;
            ...
        }
        return changed;
    }
```
写的很清楚了吧？这个方法就主要是**保存View的尺寸和位置的**。（顺带一提，View类中`getLeft()`方法返回的值就是mLeft）

然后再看看onLayout的源码

该方法有5个参数：
- boolean changed

当前View的大小和位置是不是改变了
- int l(有的可能是left，都一个意思)左边
- int t顶边
- int r右边
- int b底边

在View类中，onLayout并未做实现，因为在layout中，已经对自身View进行了位置计算，即单一View的layout过程在执行完layout()方法后就结束了，有了位置和尺寸直接画就OK，而对于ViewGroup，就需要递归布局里所有的子View，然后调用子View的`layout`就完了
```java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        //获取子View数量
        int childCount = getChildCount();
        for (int i = 0; i < childCount; i ++){

            //获取到子View
            View childView = getChildAt(i);

            //获取到子View测量得到的宽高
            int childWidth = childView.getMeasuredWidth();
            int childHeight = childView.getMeasuredHeight();

            //重新设定子View的左边、顶边、右边、底边
            int mLeft = l;
            int mTop = t;
            int mRight = mLeft + childWidth;
            int mBottom = t + childHeight;

            //调用子View的layout，保存位置
            childView.layout(mLeft, mTop, mRight, mBottom);
        }
    }
```
大概流程就是上面那样，有时候可能需要对margin做处理。

OK，这篇就到这里
