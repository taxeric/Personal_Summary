View布局贯穿在整个程序中，不管是Activity还是Fragment都提供了一个View依附的对象。那么它是如何加载的？

# 窗口管理
在Android中，平时加载View经常用setContentView实现，但View的显示并不是activity来完成的，真正控制显示的是`Window`，Window才是真正代表一个窗口。Activity相当于一个控制器，统筹视图的添加和显示，通过其他回调来与Window和View交互。

## Window
是视图的承载器，内部持有一个`DecorView`，而这个DecorView才是View的根布局。Window是一个抽象类，实际在Activity中持有的是它的子类`PhoneWindow`，在PhoneWindow中有一个内部类`DecorView`，通过创建DecorView来加载activity中设置的布局（就是R.layout.xxxxxx），Window通过WindowManager将DecorView加载到布局里面，并将DecorView交给ViewRoot，进行视图绘制以及其他交互。

## DecorView
是FrameLayout的子类，可以被认为是Android视图树的根节点视图。DecorView作为顶级视图，一般情况下它内部包含一个竖直方向的`LinearLayout`，**在这个LinearLayout里有三个部分，上面是ViewStub（根据Theme设置），下面是ContentParent（根据Theme设置），ContentParent包裹的就是ContentView**

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

