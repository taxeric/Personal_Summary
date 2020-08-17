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
getDelegate方法如下
```java
    @NonNull
    public AppCompatDelegate getDelegate() {
        if (mDelegate == null) {
            mDelegate = AppCompatDelegate.create(this, this);
        }
        return mDelegate;
    }
```
看create方法，点进去，发现其内部实现为`AppCompatDelegateImpl`
```java
    @NonNull
    public static AppCompatDelegate create(@NonNull Activity activity,
            @Nullable AppCompatCallback callback) {
        return new AppCompatDelegateImpl(activity, callback);
    }
```
在`AppCompatDelegateImpl`这个类中找到`setContentVieww(int resId)`方法，内部实现如下
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
看到个很眼熟的东西：LayoutInflater。看实现
```java
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                  + Integer.toHexString(resource) + ")");
        }

        View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
        if (view != null) {
            return view;
        }
        XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                advanceToRootNode(parser);
                final String name = parser.getName();
                ...

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    if (root != null) {
                        ...
                        // Create layout params that match root, if supplied
                        //创建布局参数给根布局
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            temp.setLayoutParams(params);
                        }
                    }
                    ...
                    // 填充所有子View
                    rInflateChildren(parser, temp, attrs, true);
                    ...
                }
            } ...

            return result;
        }
    }
```
可以看到调用了`rInflateChildren`方法来填充子View，即完成子View初始化。

