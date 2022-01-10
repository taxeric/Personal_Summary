[TOC]

# View的绘制过程

## setContentView 

`onCreate()`的`setContentView(layoutId)`代码，会生成`DecorView`、`SubDecor`，把`theme`设置给`DecorView`，将`layout`添加到`SubDecor`中，初步完成控件树的构建。

### 生成DecorView

```java
    @Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mAppCompatWindowCallback.getWrapped().onContentChanged();
    }
```

`setContentView(layoutId)`会调到`AppCompatDelegateImpl.setContentView(layoutId)`

在`ensureSubDecr()`中，会调用到`mWindow.getDecorView()`，这句代码生成了`DecorView`。`mWindow`就是`PhoneWindow`。`PhoneWindow.getDecorView()------>PhoneWindow.installDecor()`。

```java
 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            ……
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
				……
 }
```

在`installDecor()`方法中，有两个方法比较重要，一个是`generateDecor(-1)`，生成了`mDecor`，也就是`DecorView`，一个是`generateLayout(mDecor)`，读取了`theme`之后给`DecorView`进行设值，并且根据对应主题获取相应布局，`addView`设值到`DecorView`中。并且从`DecorView`中获取`id = android.R.id.content`的布局，设置为`mContentParent`。

```java
protected ViewGroup generateLayout(DecorView decor) {
  			……
        ……
				else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        mDecor.startChanging();
  			// 此处处理了mDecor.addView(layoutResource)
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        ……
        ……
}
```

`DecorView`是一个**FrameLayout**。

到这，层级结构为：

`DecorView--contentParent(android.R.id.content)`

### 生成SubDecor

接着，根据主题生成`SubDecor`，并将之前的`contentParent`的id去掉，将id设置给自己。

```java
    private ViewGroup createSubDecor() {
      	……
      	} else {
      			subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
        }
      	……
        windowContentView.setId(View.NO_ID);
        contentView.setId(android.R.id.content);
        ……
        mWindow.setContentView(subDecor);
    }
```

最后通过`mWindow.setContentView(subDecor)`将自己设置到`contentParent`中

```java
    @Override
    public void setContentView(View view, ViewGroup.LayoutParams params) {
      	……
        ……
        } else {
            // 设置到mContentParent中
            mContentParent.addView(view, params);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

`SubDecor`可能是一个ActionBarOverlayLayout，也可能是一个LinearLayout，但`ContentParent`是一个**FrameLayout**。

到这，层级结构为：

`DecorView-->contentParent(android.R.id.content)-->subDecor`

### 关联ContentView

最后执行`contentView`的添加。

```java
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```

到这，层级结构为：

`DecorView-->contentParent(android.R.id.content)-->subDecor-->contentParent-->contentView`



## handleResumeActivity

`ActivityThread`类，处理了`Activity`生命周期的相关内容。其中有关`onResume`的是`ActivityThread.handleResumeActivity`方法。其内部调用了`onRestart`、`onResume`生命周期，并将`DecorView`与`ViewRootImpl`进行了关联。

### 执行Resume

```java
 @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        // 执行生命周期onRestart/onResume
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        if (r == null) {
            // We didn't actually resume the activity, so skipping any follow-up actions.
        }
      ……
      ……
    }
```

```java
 public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        		……
            ……
            r.activity.performResume(r.startsNotResumed, reason);
   					……
   					……
 }
```

```java
 final void performResume(boolean followedByPause, String reason) {
        dispatchActivityPreResumed();
   			// 此处根据mStop的值判断是否执行onRestart
        performRestart(true /* start */, reason);

        mFragments.execPendingActions();

        mLastNonConfigurationInstances = null;

        if (mAutoFillResetNeeded) {
            // When Activity is destroyed in paused state, and relaunch activity, there will be
            // extra onResume and onPause event,  ignore the first onResume and onPause.
            // see ActivityThread.handleRelaunchActivity()
            mAutoFillIgnoreFirstResumePause = followedByPause;
            if (mAutoFillIgnoreFirstResumePause && DEBUG_LIFECYCLE) {
                Slog.v(TAG, "autofill will ignore first pause when relaunching " + this);
            }
        }

        mCalled = false;
        // 此处执行Resume方法
        mInstrumentation.callActivityOnResume(this);
 }
```

### 关联DecorView与ViewRootImpl

紧接着，会执行到创建`ViewRootImpl`并将`DecorView`设置进去的方法。

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
        ……
         if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
						……
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    // 此处设置RootViewImpl
                    wm.addView(decor, l);
                }
            }
						……
    }
```

`Activity.getWindowManager().addView()`-->`WindowManagerImpl.addView()`-->

`WindowManagerGlobal.addView()`-->`new ViewRootImpl(view.getContext(), display).setView(DecorView)`

```java
 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
            int userId) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
								……
								……
                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.
                requestLayout();
                ……
								……
                view.assignParent(this);
      					……
                ……
    }

```

在`setView`中，会先执行`requestLayout`方法，再执行`view.assignParent(this)`把`ViewRootImpl`设置为父控件。

```java
    @UnsupportedAppUsage
    void assignParent(ViewParent parent) {
        if (mParent == null) {
            mParent = parent;
        } else if (parent == null) {
            mParent = null;
        } else {
            throw new RuntimeException("view " + this + " being added, but"
                    + " it already has a parent");
        }
    }
```

为什么会先执行`requestLayout`呢。

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

```java
    @UnsupportedAppUsage
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

实际上，`requestLayout`方法是只是执行了`postCallback`方法，也就是把`mTraversalRunnable`放到了`Looper`队列中而未执行。

```
在Looper的死循环中，不断从队列中获取信息，并通过handler的dispatchMessage方法处理，而当前方法顺序执行时，实际上是在一个dispatchMessage中处理，而添加到queue队列中，会等待当前任务处理完毕，重新从queue中获取msg，再交给下一个dispatchMessage处理。
```

所以实际上，真正的测量与绘制还未执行，代码继续往下走，将`ViewRootImpl`与`DecorView`进行关联。

到这，层级结构为：

`ViewRootImpl-->DecorView-->contentParent(android.R.id.content)-->subDecor-->contentParent-->contentView`

## 常用方法

### performTraversals

当上述代码执行完毕时，执行`mTraversalRunnable`方法，最终执行到`ViewRootImpl.performTraversals`。

这个类长度近1000行，执行了一系列flag、长宽、测量、绘制等操作。

```java
// Ask host how big it wants to be
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);
```

其中，这行代码执行了第一次测量`performMeasure`。

```java
  private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
       		……
      	 	……
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
           ……
           ……
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                } else {
                    // Didn't fit in that size... try expanding a bit.
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": next baseSize="
                            + baseSize);
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        if (DEBUG_DIALOG) Log.v(mTag, "Good!");
                        goodMeasure = true;
                    }
                }
            }
        }

        if (!goodMeasure) {
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }
				……
        return windowSizeMayChange;
    }
```

之后根据情况会再次执行一到两次测量。

```java
 final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);
```

之后就执行到了`performLayout`进行布局。

```java
            // Compute new insets in place.
            mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
```

布局结束之后，会通过TreeObserver进行回调，此时，已经可以**正确获取控件的宽高**，并且也是最及时获取的地方。

之后，通过`performDraw()`执行绘制。

```java
private boolean draw(boolean fullRedrawNeeded) {
  ……
   if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
              // 执行硬件绘制
               mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
            } else {
              ……
              ……
              // 执行软件绘制
              if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
              }
            }
}
```

`performDraw()`调用`draw()`方法，最终通过硬件或者软件进行绘制。



综上，**在`onResume`中，控件还未进行测量与绘制流程，所以在其中无法获取控件的高度。**

### requestLayout

`View.requestLayout`递归到最上层，`ViewRootImpl.requestLayout`会执行`checkThread`检查线程

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

当`View`第一次执行`requestLayout`的时候，会给自己及parent递归设置**PFLAG_FORCE_LAYOUT**，执行完layout之后，去掉**PFLAG_FORCE_LAYOUT**

第二次调用`View.requestLayout`时，检查parent是否设置了**PFLAG_FORCE_LAYOUT**，如果是，则不再递归调用`parent.requestLayout`

```java
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();

        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
            // Only trigger request-during-layout logic if this is the view requesting it,
            // not the views in its parent hierarchy
            ViewRootImpl viewRoot = getViewRootImpl();
            if (viewRoot != null && viewRoot.isInLayout()) {
                if (!viewRoot.requestLayoutDuringLayout(this)) {
                    return;
                }
            }
            mAttachInfo.mViewRequestingLayout = this;
        }
				// 设置flag
        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;
				// 判断是否调用父控件的requestLayout方法
        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
            mAttachInfo.mViewRequestingLayout = null;
        }
    }
```

执行`requestLayout()`，检查线程，最终调用`performTraversals()`时，`mLayoutRequested`设置为true，所以会执行`measureHierarchy()`、`performLayout()`和`performDraw()`。

#### measure

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
       	……
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
				……
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
           	……
            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
						……
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
			……
    }
```

`measureHierarchy()`最终调用`View.measure()`方法，其中，设置了**PFLAG_FORCE_LAYOUT**及尺寸发生变化的View，会执行`onMeasure()`进行**测量**。执行完测量之后，会添加上**PFLAG_LAYOUT_REQUIRED**。

#### layout

```java
 public void layout(int l, int t, int r, int b) {
   			……
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);
        }
   			……
 }

```

在`View.layout()`方法中，设置了**PFLAG_LAYOUT_REQUIRED**及四个顶点发生过改变的View，会执行`onlayout()`方法进行布局。



所以，对于调用了`requestLayout()`方法的View，其**自身及父View**一定会执行`onMeasure()`及`onLayout()`方法，而子View，则需要**判断其尺寸是否发生变化**。

#### draw

```java
private boolean draw(boolean fullRedrawNeeded) {
				……
        if (fullRedrawNeeded) {
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        }
				……
        boolean accessibilityFocusDirty = false;
        final Drawable drawable = mAttachInfo.mAccessibilityFocusDrawable;
        if (drawable != null) {
            final Rect bounds = mAttachInfo.mTmpInvalRect;
            final boolean hasFocus = getAccessibilityFocusedRect(bounds);
            if (!hasFocus) {
                bounds.setEmpty();
            }
            if (!bounds.equals(drawable.getBounds())) {
                accessibilityFocusDirty = true;
            }
        }
				……
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
              	// 开启硬件加速
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
            } else {
               // 关闭硬件加速
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
                }
            }
        }
				……
        return useAsyncReport;
    }
```

`draw()`需要执行绘制，需要满足`dirty`不为空，而前面`fullRedrawNeeded`为true时，会给`dirty`赋值。

```java
        if (fullRedrawNeeded) {
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        }
```

```java
 private void performTraversals() {
   ……
				else {
          // 非第一次执行
            desiredWindowWidth = frame.width();
            desiredWindowHeight = frame.height();
            if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
                if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
                mFullRedrawNeeded = true;
                mLayoutRequested = true;
                windowSizeMayChange = true;
            }
        }
   ……
     
```

非第一次执行，并且`Window`的尺寸改变时，会让`mFullRedrawNeeded = true`。

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
 }
```

或者，在执行`View.layout()`时，判断当前View尺寸是否改变，会调用`setFrame()`方法

```java
protected boolean setFrame(int left, int top, int right, int bottom) {
        ……
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            changed = true;

            // Remember our drawn bit
            int drawn = mPrivateFlags & PFLAG_DRAWN;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);
        }
}
```

进而调到`invalidate(sizechanged)`方法，递归调用到`ViewRootImpl.invalidate()`方法，为`mDirty`赋值。

```java
    @UnsupportedAppUsage
    void invalidate() {
        mDirty.set(0, 0, mWidth, mHeight);
        if (!mWillDrawSoon) {
            scheduleTraversals();
        }
    }
```

所以，绘制也不一定会执行，得看View的尺寸是否改变。

### invalidate

调用`invalidate()`刷新时，递归调用`parent.invalidateChildren`，调用到

* 如果开启了硬件加速
  * 最终直接调用`ViewRootImpl.scheduleTraversals`触发绘制，绕过了检查了线程的方法，执行`ViewRootImpl.scheduleTraversals`触发绘制。
  * 只有需要重绘的View设置了**PFLAG_INVALIDATED**，仅绘制它自己，其它布局没有此Flag，不参与重绘。
* 如果关闭了硬件加速
  * 最终调用到`ViewRootImpl.invalidateChildInParent`，并检查线程，执行`ViewRootImpl.scheduleTraversals`触发绘制。
  * 会重绘自DecorView往下所有布局。

在调用绘制的`performTraversals()`方法中，`mLayoutRequested`没有被赋值，于是绕过了测量`measureHierarchy()`与布局`performLayout()`方法，只执行`performDraw()`方法进行绘制。

```java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```

为`mLayoutRequested`赋值的`requestLayout()`方法。

### requestLayout与invalidate的异同

#### 相同点

* 最终都会调用到`ViewRootImpl.performLayout()`方法。

#### 差异点

* `requestLayout`
  * 会依次执行`performLayout()`中的`measureHierarchy()`、`performLayout()`和`performDraw()`方法，对于**自己及父控件**，会执行`onMeasure()`与`onLayout()`方法，只有尺寸改变时，会执行`onDraw()`方法。**子View**需要判断尺寸是否改变，才执行`onMeasure()`及`onLayout()`。
* `invalidate`
  * 不会执行`performLayout()`中的`measureHierarchy()`、`performLayout()`方法，只执行`performDraw()`，也就是不会执行`onMeasure()`及`onLayout()`方法，只执行`onDraw()`。
  * 开启了硬件加速则只绘制调用者View
  * 关闭硬件加速则绘制`DecorView`下所有View。

所以，需要重新测量与布局，就调用`requestLayout()`，如果需要重新绘制，则调用`invalidate()`。

## 渲染

* 屏幕刷新率

  一秒内屏幕刷新的次数，比如常见的60hz，就是一秒内刷新60次。

* 逐行扫描

  屏幕每次显示画面，都是从左到右，从上到下逐行扫描，按顺序显示像素点的。以刷新率60hz为例，一秒内刷新60次，则每次刷新消耗的时间1000/60 ≈ 16.6ms，就是一次逐行扫描花费的时间。

* 帧率

  表示GPU在一秒内绘制的帧数，单位是fps。帧率是动态的，以60fps为例，一秒钟内GPU做多可以绘制60帧画面。而帧率是动态的，如果画面没有变化，则屏幕读取的还是上次GPU保存在buffer中的数据。

* VSync

  以固定的频率进行通知的信号，表示当前最佳的渲染时间。以刷新率60hz为例，每16.6ms触发一次VSync。CPU/GPU的绘制是在VSYNC到来时开始的。

* Choreographer

  一个协调动画、输入和绘图的计时java类。在VSync信号通知时执行。



### Choreographer的创建

所有的UI变化，都会走到`ViewRootImpl`类中的`scheduleTraversals()`方法。

```java
    @UnsupportedAppUsage
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            // 同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            // choregrapher的runnable回调，最终执行到doTraversal方法。
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
		final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
    final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            //移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            ...
            //开始三大绘制流程
            performTraversals();
            ...
        }
    }
```

callback中的`mTraversalRunnable`，会在下一个VSync到来之后执行，并触发`doTraversal`、`performTraversals`及绘制流程。

`mChoreographer`会在`ViewRootImpl`的构造方法中被创建

```java
mChoreographer = useSfChoreographer
        ? Choreographer.getSfInstance() : Choreographer.getInstance();
```

```java
public static Choreographer getInstance() {
    return sThreadInstance.get();
}
```

```java
private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
      	// 获取主线程Looper并传入构造方法
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
    }
};
```

```java
public static final int CALLBACK_COMMIT = 4;

private static final int CALLBACK_LAST = CALLBACK_COMMIT;

// Choreographer的构造方法
private Choreographer(Looper looper, int vsyncSource) {
        mLooper = looper;
  			// 使用当前线程的looper创建handler
        mHandler = new FrameHandler(looper);
  			// 4.1以上USE_VSYNC默认为true
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;
				// 获取一帧的时间 如果屏幕刷新率是60hz，则为16.6ms
        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
				// 创建一个size = 5 的链表
        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
          	// 创建回调队列，放入链表
            mCallbackQueues[i] = new CallbackQueue();
        }
        // b/68769804: For low FPS experiments.
        setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

在上述创建的回调链表中，会保存有五种类型的回调，如下

```java
		//输入事件，首先执行
    public static final int CALLBACK_INPUT = 0;
    //动画，第二执行
    public static final int CALLBACK_ANIMATION = 1;
    //插入更新的动画，第三执行
    public static final int CALLBACK_INSETS_ANIMATION = 2;
    //绘制，第四执行
    public static final int CALLBACK_TRAVERSAL = 3;
    //提交，最后执行，
    public static final int CALLBACK_COMMIT = 4;
```

每次收到VSync信号时，会依次执行这五种任务，第一个执行INPUT任务，最后执行COMMIT任务。



### 添加ChoreoGrapher回调

```java
mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```

在`scheduleTraversals`设置回调任务中，传入的类型是`Choreographer.CALLBACK_TRAVERSAL`绘制，会在第四个执行。

调用`postCallback`之后，会执行到`postCallbackDelayedInternal（）`方法。

```java
private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    if (DEBUG_FRAMES) {
        Log.d(TAG, "PostCallback: type=" + callbackType
                + ", action=" + action + ", token=" + token
                + ", delayMillis=" + delayMillis);
    }

    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
      	// 根据类型，保存到对应类型的链表中，并且记录执行时间
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
				
        if (dueTime <= now) {
          	// 当执行时间已经错过时，会直接调用scheduleFrameLocked
            scheduleFrameLocked(now);
        } else {
          	// 否则添加一个异步任务
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}
```

这个添加异步任务的`mHandler`就是之前构造方法中创建的`FrameHandler`。

```java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                		// 立即绘制
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                		// 申请VSync信号
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    // 发送过来的异步任务，
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
    void doScheduleCallback(int callbackType) {
        synchronized (mLock) {
            if (!mFrameScheduled) {
                final long now = SystemClock.uptimeMillis();
              	// 判断当前是否到了此类型队列的执行时间，callbackType为当初调用postCall传入的类型
              	// 此处为Choreographer.CALLBACK_TRAVERSAL
                if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
                    scheduleFrameLocked(now);
                }
            }
        }
    }
```

会发现最后这里执行的方法与最开始postCallbackDelayedInternal中到时间后执行的方法一致。均为`scheduleFrameLocked()`。

```java
 private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
          	// 4.1之后默认为true
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // 如果当前执行在Loop的线程，则立即申请VSync信号
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                // 否则切换到Loop线程，尽快执行申请VSync信号
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
              	// 4.1之前
              	// 计算下一帧所在时间点
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
              	// 到达时间之后，发送MSG_DO_FRAME信号，直接执行doFrame方法
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```

这边看到，所有的动作，最后都是为了**申请下一个VSync信号**。



### 申请VSync信号

```java
    @UnsupportedAppUsage
    private void scheduleVsyncLocked() {
        // 通过DisplayEventReceiver申请Vsync
      	// mDisplayEventReceiver在我们初始化Choreographer的时候创建
        mDisplayEventReceiver.scheduleVsync();
    }

    public DisplayEventReceiver(Looper looper, int vsyncSource, int configChanged) {
        if (looper == null) {
            throw new IllegalArgumentException("looper must not be null");
        }

        mMessageQueue = looper.getQueue();
      	// 在父类构造方法中构造监听
        mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue, vsyncSource, configChanged);

        mCloseGuard.open("dispose");
    }

    @UnsupportedAppUsage
    public void scheduleVsync() {
      	// 此方法实现在父类DisplayEventReceiver中
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
          	// 通过native方法进行申请信号，并回调onVsync方法
            nativeScheduleVsync(mReceiverPtr);
        }
    }
```

当Vsync信号来临时，触发`FrameDisplayEventReceiver`中的`onVsync()`方法。

```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    private boolean mHavePendingVsync;
    private long mTimestampNanos;
    private int mFrame;

    public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
        super(looper, vsyncSource, CONFIG_CHANGED_EVENT_SUPPRESS);
    }

    // TODO(b/116025192): physicalDisplayId is ignored because SF only emits VSYNC events for
    // the internal display and DisplayEventReceiver#scheduleVsync only allows requesting VSYNC
    // for the internal display implicitly.
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        // Post the vsync event to the Handler.
        // The idea is to prevent incoming vsync events from completely starving
        // the message queue.  If there are no messages in the queue with timestamps
        // earlier than the frame time, then the vsync event will be processed immediately.
        // Otherwise, messages that predate the vsync event will be handled first.
        long now = System.nanoTime();
        if (timestampNanos > now) {
        // 检查当前Vsync时间是否正确
            Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                    + " ms in the future!  Check that graphics HAL is generating vsync "
                    + "timestamps using the correct timebase.");
            timestampNanos = now;
        }

        if (mHavePendingVsync) {
            Log.w(TAG, "Already have a pending vsync event.  There should only be "
                    + "one at a time.");
        } else {
            mHavePendingVsync = true;
        }
				// 发送异步任务，异步任务为执行doFrame方法，执行时间为当前
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }
}
```

当VSync到来时，将`doFrame`任务添加到异步队列。

此时，`doFrame`任务并不是立即执行，如果队列中还有其他任务在前面，会先执行它们。



### 执行doFrame

当信号来临时，最终会执行`doFrame`方法。

```java
@UnsupportedAppUsage
void doFrame(long frameTimeNanos, int frame) {
    final long startNanos;
    synchronized (mLock) {
        if (!mFrameScheduled) {
            return; // no work to do
        }

        if (DEBUG_JANK && mDebugPrintNextFrameTimeDelta) {
            mDebugPrintNextFrameTimeDelta = false;
            Log.d(TAG, "Frame time delta: "
                    + ((frameTimeNanos - mLastFrameTimeNanos) * 0.000001f) + " ms");
        }
				// 任务的预期执行时间
        long intendedFrameTimeNanos = frameTimeNanos;
      	// 执行到doFrame的时间
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
          	// 如果执行到doFrame的时间与预期时间之差，大于1帧的时间（16.6ms）
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
          	// 则计算具体帧数，如果大于30帧，则打印信息，提示主线程执行太多任务导致卡顿。
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
          	// 按时间差取余，得到上一帧到当前的偏移时间
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            if (DEBUG_JANK) {
                Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                        + "which is more than the frame interval of "
                        + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                        + "Skipping " + skippedFrames + " frames and setting frame "
                        + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
            }
          	// 计算上一次执行帧的时间
            frameTimeNanos = startNanos - lastFrameOffset;
        }

      	// 如果时间不对，则等待下一帧的到来。
        if (frameTimeNanos < mLastFrameTimeNanos) {
            if (DEBUG_JANK) {
                Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                        + "previously skipped frame.  Waiting for next vsync.");
            }
            scheduleVsyncLocked();
            return;
        }

        if (mFPSDivisor > 1) {
            long timeSinceVsync = frameTimeNanos - mLastFrameTimeNanos;
            if (timeSinceVsync < (mFrameIntervalNanos * mFPSDivisor) && timeSinceVsync > 0) {
                scheduleVsyncLocked();
                return;
            }
        }

        mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
        mFrameScheduled = false;
      	// 记录最后一帧的时间
        mLastFrameTimeNanos = frameTimeNanos;
    }

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
        AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);
				// 按时间顺序执行回调
        mFrameInfo.markInputHandlingStart();
      	// 处理INPUT事件
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

        mFrameInfo.markAnimationsStart();
      	// 处理动画
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
      	// 处理插入动画
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);

        mFrameInfo.markPerformTraversalsStart();
      	// 处理绘制
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
      	// 处理COMMIT
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    } finally {
        AnimationUtils.unlockAnimationClock();
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }

    if (DEBUG_FRAMES) {
        final long endNanos = System.nanoTime();
        Log.d(TAG, "Frame " + frame + ": Finished, took "
                + (endNanos - startNanos) * 0.000001f + " ms, latency "
                + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
    }
}
```

最终处理回调方法`doCallbacks（）`

```java
void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            // We use "now" to determine when callbacks become due because it's possible
            // for earlier processing phases in a frame to post callbacks that should run
            // in a following phase, such as an input event that causes an animation to start.
            final long now = System.nanoTime();
          	// 拿到所有callback
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;

            // Update the frame time if necessary when committing the frame.
            // We only update the frame time if we are more than 2 frames late reaching
            // the commit phase.  This ensures that the frame time which is observed by the
            // callbacks will always increase from one frame to the next and never repeat.
            // We never want the next frame's starting frame time to end up being less than
            // or equal to the previous frame's commit frame time.  Keep in mind that the
            // next frame has most likely already been scheduled by now so we play it
            // safe by ensuring the commit time is always at least one frame behind.
            if (callbackType == Choreographer.CALLBACK_COMMIT) {
              	// 如果此回调是COMMIT，则记录当前时间与执行帧的时间差
                final long jitterNanos = now - frameTimeNanos;
                Trace.traceCounter(Trace.TRACE_TAG_VIEW, "jitterNanos", (int) jitterNanos);
                if (jitterNanos >= 2 * mFrameIntervalNanos) {
                  	// 如果时间差超过2帧的时间，则获取偏移量
                    final long lastFrameOffset = jitterNanos % mFrameIntervalNanos
                            + mFrameIntervalNanos;
                    if (DEBUG_JANK) {
                        Log.d(TAG, "Commit callback delayed by " + (jitterNanos * 0.000001f)
                                + " ms which is more than twice the frame interval of "
                                + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                                + "Setting frame time to " + (lastFrameOffset * 0.000001f)
                                + " ms in the past.");
                        mDebugPrintNextFrameTimeDelta = true;
                    }
                  	// 记录上一帧应该在的时间
                    frameTimeNanos = now - lastFrameOffset;
                    mLastFrameTimeNanos = frameTimeNanos;
                }
            }
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "RunCallback: type=" + callbackType
                            + ", action=" + c.action + ", token=" + c.token
                            + ", latencyMillis=" + (SystemClock.uptimeMillis() - c.dueTime));
                }
              	// 拿到callback，执行其run方法
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

    private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        public Object action; // Runnable or FrameCallback
        public Object token;

        @UnsupportedAppUsage
        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
              	// 通过postFrameCallback 或 postFrameCallbackDelayed，会执行这里
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
              	// ViewRootImpl中的token == null，所以执行的是run方法
                ((Runnable)action).run();
            }
        }
    }
```

最终，执行了当初`ViewRootImpl`中传入的`callback`，执行绘制。

```java
mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
```



### 计算丢帧

**Choreographer的postFrameCallback()通常用来计算丢帧情况**，使用方式如下：

```java
		//Application.java
         public void onCreate() {
             super.onCreate();
             //在Application中使用postFrameCallback
             Choreographer.getInstance().postFrameCallback(new FPSFrameCallback(System.nanoTime()));
         }


    public class FPSFrameCallback implements Choreographer.FrameCallback {

      private static final String TAG = "FPS_TEST";
      private long mLastFrameTimeNanos = 0;
      private long mFrameIntervalNanos;

      public FPSFrameCallback(long lastFrameTimeNanos) {
          mLastFrameTimeNanos = lastFrameTimeNanos;
          mFrameIntervalNanos = (long)(1000000000 / 60.0);
      }

      @Override
      public void doFrame(long frameTimeNanos) {

          //初始化时间
          if (mLastFrameTimeNanos == 0) {
              mLastFrameTimeNanos = frameTimeNanos;
          }
          final long jitterNanos = frameTimeNanos - mLastFrameTimeNanos;
          if (jitterNanos >= mFrameIntervalNanos) {
              final long skippedFrames = jitterNanos / mFrameIntervalNanos;
              if(skippedFrames>30){
              	//丢帧30以上打印日志
                  Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                          + "The application may be doing too much work on its main thread.");
              }
          }
          mLastFrameTimeNanos=frameTimeNanos;
          //注册下一帧回调
          Choreographer.getInstance().postFrameCallback(this);
      }
  }

```

## Handler的同步屏障

在Handler中，信息分为三种，同步信息、异步信息和同步屏障。

```java
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

在Handler的构造方法中，有一个boolean类型的入参`async`，表示是否是异步消息。

```java
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

最终在进入消息队列时，通过`msg.setAsynchronous`方法设置异步标记。



### 插入同步屏障

消息屏障，通过队列的`postSyncBarrier（）`方法插入。

```java
private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
          	// 将消息插入到正确的时间---找到距离when最近的消息，并将prev指向它
            while (p != null && p.when <= when) {
              	// 如果当前信息的时间小于when，屏障指定时间，则将prev指针指向当前消息
                prev = p;
              	// 获取当前消息的下一个消息
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
          	// 如果找到prev，则将屏障插入prev与prev的下一个信息之间
            msg.next = p;
            prev.next = msg;
        } else {
          	// 否则将屏障插入当前信息之前
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

* 屏障与普通信息的区别在于，没有**target**属性。普通信息需要将msg发送给`target`，而屏障不用。

- **屏障和普通消息一样可以根据时间来插入到消息队列中的适当位置，并且只会挡住它后面的同步消息的分发**

* postSyncBarrier()返回一个int类型的数值token，通过这个数值可以撤销屏障即removeSyncBarrier()。
* 插入普通消息会唤醒消息队列，但是插入屏障不会。



### 阻拦同步消息

```java
Message next() {
   	……
    for (;;) {
        ……
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                // 如果没有target，则表示是同步屏障
                do {
                		// 遍历队列，获取下一个异步消息
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            ……
    }
}
```

