View布局贯穿在整个程序中，不管是Activity还是Fragment都提供了一个View依附的对象。那么它是如何加载的？

# 窗口管理
在Android中，平时加载View经常用setContentView实现，但View的显示并不是activity来完成的，真正控制显示的是`Window`，Window才是真正代表一个窗口。Activity相当于一个控制器，统筹视图的添加和显示，通过其他回调来与Window和View交互。

## Window
是视图的承载器，内部持有一个`DecorView`，而这个DecorView才是View的根布局。Window是一个抽象类，实际在Activity中持有的是它的子类`PhoneWindow`，在PhoneWindow中有一个内部类`DecorView`，通过创建DecorView来加载activity中设置的布局（就是R.layout.xxxxxx），Window通过WindowManager将DecorView加载到布局里面，并将DecorView交给ViewRoot，进行视图绘制以及其他交互。

## DecorView
是FrameLayout的子类，可以被认为是Android视图树的根节点视图。DecorView作为顶级视图，一般情况下它内部包含一个竖直方向的`LinearLayout`，**在这个LinearLayout里有三个部分，上面是ViewStub（根据Theme设置），下面是ContentParent（根据Theme设置），ContentParent包裹的就是ContentView**

# 源码分析

众所周知，在打开Activity时，会执行setContent(R.layout.xxx)方法，点进去可以看到如下代码（本文以 API 29 为例）
```java
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
```
先不看是怎么实现的，先看看它的父类：Activity里面是怎么写的
```java
    /**
     * 给activity设置xml布局，把所有顶级视图添加到这个activity
     */
    public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```
不知道什么是**顶级视图**，接着看源码。这个方法调用了一个`getWindow()`方法，点进去
```java
    public Window getWindow() {
        return mWindow;
    }
```
就这....看注释也看不出是啥，就先看看Window是啥。点进去可以看到Window是个抽象类
```java
public abstract class Window {
    ...
}
```
注释：用于**顶级窗口**外观和行为策略的基类。该类的实例应被作为顶级视图，添加到WindowManager，它提供标准的UI策略，例如背景、标题等。它只有一个实现类：PhoneWindow

额..那就能得出getWindow获取到的是`PhoneWindow`的`setContentView`方法。跳到PhoneWindow，结果发现这个类爆红，推测可能是有hide注解的所以看不了，就看看是别人怎么写的：
（以下全部是别人的总结）
```java
  @Override
  public void setContentView(int layoutResID) {
      // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
      // decor, when theme attributes and the like are crystalized. Do not check the feature
      // before this happens.
      if (mContentParent == null) {
          installDecor();
      } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          mContentParent.removeAllViews();
      }
      if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
          final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                  getContext());
          transitionTo(newScene);
      } else {
          mLayoutInflater.inflate(layoutResID, mContentParent);
      }
      mContentParent.requestApplyInsets();
      final Callback cb = getCallback();
      if (cb != null && !isDestroyed()) {
          cb.onContentChanged();
      }
      mContentParentExplicitlySet = true;
  }
```
第一个判断推测是mContentParent初始化的逻辑，下面的判断是把id为layoutResId的资源加载到mContentParent里，OK先看到这里，看看mContentParent是个啥：
```java
    // This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    ViewGroup mContentParent;
```
注释：放置Window的View，可能是DecorView，也可能是DecorView的子View

又出来个`DecorView`的概念，既然在上面的setContentView方法中有个`installDecor`方法，那就点进去看看这个方法干了啥：
```java
    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

        ///...  省略若干代码
        }
    }
```
可以看到在mContent初始化前，一个叫`mDecor`的东西初始化了。看看这是个啥：
```java
    // This is the top-level view of the window, containing the window decor.
    private DecorView mDecor;
```
注释：这是Window的顶层视图，包含Window的装饰

看看DecorView代码
```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
  //... 省略很多很多代码
}
```
可以得出：DecorView是一个FrameLayout，是最顶层的View。包含Window（顶级窗口）的装饰（比如大小等属性）会体现在这个View上。

接着回到PhoneWindow，看看`mDecor`是怎么初始化的
```java
   protected DecorView generateDecor(int featureId) {
      // System process doesn't have application context and in that case we need to directly use
      // the context we have. Otherwise we want the application context, so we don't cling to the
      // activity.
      Context context;
      if (mUseDecorContext) {
          Context applicationContext = getContext().getApplicationContext();
          if (applicationContext == null) {
              context = getContext();
          } else {
              context = new DecorContext(applicationContext, getContext().getResources());
              // 设置主题
              if (mTheme != -1) {
                  context.setTheme(mTheme);
              }
          }
      } else {
          context = getContext();
      }
      // new 一个DecorVIew
      return new DecorView(context, featureId, this, getAttributes());
  }
```
在最后一行代码中，`DecorView`的第三个参数就是`PhoneWindow`，即初始化DecorView时会把PhoneWindow传进去。再看`installDecor`方法
```java
    private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            ...
        } else {
            mDecor.setWindow(this);
        }
        ...
    }
```
当mDecor不为空时，setWindow方法传入的参数也是当前PhoneWindow。在DecorView类中，能看到类似如下方法
```java
public void setWindowBackground(Drawable drawable) {
        if (getBackground() != drawable) {
            setBackgroundDrawable(drawable);
            if (drawable != null) {
                mResizingBackgroundDrawable = enforceNonTranslucentBackground(drawable,
                        mWindow.isTranslucent() || mWindow.isShowingWallpaper());
            } else {
                mResizingBackgroundDrawable = getResizingBackgroundDrawable(
                        getContext(), 0, mWindow.mBackgroundFallbackResource,
                        mWindow.isTranslucent() || mWindow.isShowingWallpaper());
            }
            if (mResizingBackgroundDrawable != null) {
                mResizingBackgroundDrawable.getPadding(mBackgroundPadding);
            } else {
                mBackgroundPadding.setEmpty();
            }
            drawableChanged();
        }
    }
```
或者
```java
    final WindowManager.LayoutParams attrs = mWindow.getAttributes();
    if ((attrs.flags & FLAG_LAYOUT_IN_SCREEN) == 0) {
        if (attrs.height == WindowManager.LayoutParams.WRAP_CONTENT) {
        }
    }
```
可以看到DecorView是根据window的属性来设置自己的属性的。

回到PhoneWindow，在installDecor方法里面初始化mDecor后，会接着初始化mContentParent，如下：
```java
generateLayout(mDecor);
```
看看是怎么实现的
```java
 protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        // 拿window设置的Style
        TypedArray a = getWindowStyle();

        // 是不是浮动的Window ，例如Dialog等
        mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }
// ... ... 省略若干代码。
}
```
可以看到第一句就是获取style，TypedArray在 笔记0 里面已经说了，是用于获取自定义控件的属性的，可以看出PhoneWindow也是这么做的。继续看下面的代码
```java
  // 获取window的属性
   TypedArray a = getWindowStyle();

  // 是不是浮动的Window ，例如Dialog等
  mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        
  // 是不是设置了noTitle的style
  if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {}
   
  // 是不是透明的Window
  mIsTranslucent = a.getBoolean(R.styleable.Window_windowIsTranslucent, false);
  ```
  这些代码就是从TypedArray获取值来判断：
  - window是不是可浮动的，如dialog
  - 判断是否设置了Title
  - 判断是不是透明的window
  - ....
  
  然后把这些值赋给PhoneWindow的成员变量。接着往下看：
  ```java
      int layoutResource;
    // 拿到设置属性，然后去加载不同的XML。
    int features = getLocalFeatures();
    // System.out.println("Features: 0x" + Integer.toHexString(features));
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    } else if (){
        // 省略若干代码... ...
    } else {
        // Embedded, so no decoration is needed.
        layoutResource = R.layout.screen_simple;
        // System.out.println("Simple!");
    }
   // 将加载的布局文件加载到DecorView中去 
       mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
```
基本理解，最后会把layoutResource（也就是布局）加载到DecorView中。看看onResourcesLoaded是怎么实现的
```java
  void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
       mStackId = getStackId();
      if (mBackdropFrameRenderer != null) {
          loadBackgroundDrawablesIfNeeded();
          mBackdropFrameRenderer.onResourcesLoaded(
                  this, mResizingBackgroundDrawable, mCaptionBackgroundDrawable,
                  mUserCaptionBackgroundDrawable, getCurrentColor(mStatusColorViewState),
                  getCurrentColor(mNavigationColorViewState));
      }
      mDecorCaptionView = createDecorCaptionView(inflater);
      final View root = inflater.inflate(layoutResource, null);
      if (mDecorCaptionView != null) {
          if (mDecorCaptionView.getParent() == null) {
              addView(mDecorCaptionView,
                      new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
          }
          mDecorCaptionView.addView(root,
                  new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
      } else {
          // Put it below the color views.
          addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
      }
      mContentRoot = (ViewGroup) root;
      initializeElevation();
  }
```
看到首先把layoutReource加载成一个View，然后用addView加载到mDecor里。在加载之前，会对`mDecorCaptionView`做判断，看看它是怎么初始化的：
```java
    // Free floating overlapping windows require a caption.
    private DecorCaptionView createDecorCaptionView(LayoutInflater inflater){
        // ... 省略若干代码
    }
```
注释：自由浮动的重叠窗口需要一个标题

emmm，不明白什么意思，就不看了....只看它为空的情况，如下：
```java
  final View root = inflater.inflate(layoutResource, null);
  if (...){
    ...
  } else {
    addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
  }
```
OK，看看到底加载了啥样的布局到mDecor里面。往上看，在根据window属性设置decorView属性时会加载不同布局，看最后一个布局：R.layout.screen_simple
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
这个布局就不用多说了吧。往回接着看`generateLayout`的逻辑：
```java
    // 找到刚刚的Content
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
    // ... ... 省略若干代码
    // 最后返回这个contentParent
    return contentParent;
```
可以看到`findViewById(ID_ANDROID_CONTENT)`，看看这个`ID_ANDROID_CONTENT`是啥
```java
    /**
     * The ID that the main layout in the XML layout file should have.
     */
    public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```
到这里就知道了，ID_ANDROID_CONTENT就是上面xml布局里FrameLayout的id

OK，到这里就看完`installDecor`方法了，总结一下干了些啥：
1. 初始化mDecor
2. 初始化mContentParent，包括：获取window的属性并赋给PhoneWindow，根据window的属性加载不同布局，然后加载到DecorView，最后通过findViewById返回mContentParent

回过头看PhoneWindow的setContentView：
```java
 @Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
            //不为空诗先RemoviewAllViews
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        // 是否存在动画
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            // 把资源文件添加到 mContentParent 中去
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```
能看到
```java
mLayoutInflater.inflate(layoutResID, mContentParent);
```
就是把自己写的布局加载到mContentParent里面
## 流程回顾
1. 初始化mDecor
```java
generateDecor(-1)
```
2. 初始化mContentParent
```java
generateLayout(mDecor);
```
3. 把布局加载到mContentParent
```java
 public void setContentView(int layoutResID) {
       mLayoutInflater.inflate(layoutResID, mContentParent);
 }
```
整个层级结构如下：
- activity 包含 window（就是PhoneWindow）
- window 包含 DecorView
- DecorView 包含 一个LinearLayout（里面有两个组件，一个ViewStub，一个mContentParent，前者是标题，后者id为R.id.content，用于填充自己写的xml）

