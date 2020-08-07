View布局贯穿在整个程序中，不管是Activity还是Fragment都提供了一个View依附的对象。那么View是如何加载的？

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
```


