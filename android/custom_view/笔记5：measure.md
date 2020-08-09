在布局中，每个View的大小不仅取决于自身，也受父控件影响。`measure`用于View的自我测量，虽然是这么说，不过真正进行自我测量的是方法内部的`onMeasure`方法。`measure`是一个调度方法，会做测量前的预处理操作，`onMeasure`主要就是测量View的大小及模式，举个例子
```java
    //ViewGroup中测量子View的方法，可以看到最后调用了子View的measure方法
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
测量过程分两种情况考虑：
- 子View就是一个普通的View；
- 子View是一个ViewGroup。

# View的Measure
如果子View就是一个View，那么该方法干的事就一个：计算和保存自己的尺寸。笔记1咱就是这么干的：
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        if (widthMode == MeasureSpec.AT_MOST && heightMode == MeasureSpec.AT_MOST){
            setMeasuredDimension(200, 200);
        }else if (widthMode == MeasureSpec.AT_MOST){
            ...
        }...
    }
```
`setMeasureDimension`方法是用来保存尺寸的，此方法必须被调用在`onMeasure`中。这里根据测量模式重新设定值，保存值

## 深入一点
在笔记1的第一个问题中，由于没有重写`onMeasure`方法，其父类实现为
```java
//测量视图及其内容，以确定测量的宽度和测量的高度。 
//measure(int，int)方法将被调用，并且子类应重写此方法以提供对其内容的准确和有效的度量。

//重写此方法时，必须调用 setMeasuredDimension(int，int)来存储此视图的测量宽度和高度。 
//否则将触发由measureint，int)引发的IllegalStateException。

//如果重写此方法，则子类负责确保所测量的高度和宽度至少为视图的最小高度和宽度
//(getSuggestedMinimumHeight()和getSuggestedMinimumWidth())

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
这里的两个参数：widthMeasureSpec、heightMeasureSpec，指的是`父控件传递给当前子控件的一个建议值（关于这点可以在上面ViewGroup的measureChild方法看出来），即想把当前子控件宽高设为该参数值`，**不是**父控件的宽高**也不是**子控件的宽高，实际尺寸还是得看最后你保存的是多少

`getDefaultSize`方法如下
```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```
这里可以看出，当未重写`onMeasure`方法(或重写了该方法但在方法内没有调用`setMeasuredDimension`方法)时，`AT_MOST`与`EXACTLY`模式的值一致，即**未重写onMeasure方法时，warp_content与match_parent效果一致**

## Question
为什么子控件布局设置warp_content也是填充父容器？

## Answer
从getDefaultSize方法可以看出，specSize值来自MeasureSpec.getSize(measureSpec)，故此问题在于子控件的specSize值是多少。specSize值是来自onMeasure方法的参数，向上找，可以看出`measure`方法中传递了这个参数，再看看`measure`方法被谁调用了，ctrl+点击，可以看到一堆调用，不过仔细看，调用这个方法的类大都像是布局控件，像Toolbar之类的，而其父类是`ViewGroup`，往下找也能看到在ViewGroup类中也调用了measure方法，所以看看ViewGroup类中这个方法是咋被调用的。点进去发现调用`measure`方法的有俩：`measureChild()`和`measureChildWithMargins()`，看名字就能才出来是干啥用的，看看measureChild方法
```java
    //ViewGroup的方法，可以看到最后调用了子View的measure方法
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
很清楚了吧，我们能得到两个结论：这里的`childWidthMeasureSpec`和`childHeightMeasureSpec`值是由`getChildMeasureSpec`方法得出来的；measure方法是由父View计算子View视图大小时被调用的

点进去`getChildMeasureSpec`方法，看得出来....逻辑有点棘手啊，不过仔细看看还是中规中矩的。简要代码如下
```java
//这里的spec为父ViewGroup的MeasureSpec值
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //父ViewGroup的测量模式和测量大小
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;
    //判断父ViewGroup的测量模式
    switch (specMode) {
    //当父ViewGroup为match_parent时
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            //这个判断比较简单
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            //如果子View布局为warp_content，则子View最终布局大小为specSize，
            //即父ViewGroup布局大小，子View最终测量模式为AT_MOST
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
        ...
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
OK，看到这里就能解释为什么当子控件布局为warp_content时，最终显示与match_parent效果一致了。而且咱又得出一结论：子控件的specMode和specSize值不仅由自身的LayoutParams决定，也由父控件的MeasureSpec决定。在笔记1中，父控件(容器)为LinearLayout，其根布局参数为match_parent。最后将该逻辑总结如下

![image](https://img-blog.csdnimg.cn/20200727143647717.png)

下面的是别人的总结

![image](https://upload-images.jianshu.io/upload_images/944365-6088d2d291bbae09.png?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)

一般情况下，控件的建议尺寸和实际尺寸一致

## Question
问题又来了，最顶层View的MeasureSpec是由谁决定？
## Answer
这里采用别人的一段话：

在Android中，所有视图（Activity、Dialog等）都是`Window`，由笔记3可知，DecorVieiw是Activity的根布局，传递给DecorView的MeasureSpec是系统根据Activity或Dialog的Theme来确定的，也就是说，最初的MEasureSpec是直接根据Window的属性构建的，一般对于Activity来说，根MeasureSpec是EXACTLY+屏幕尺寸，对于Dialog来说，如果不做特殊设定会采用AT_MOST+屏幕尺寸

# ViewGroup的Measure
如果子View是一个ViewGroup，那么子View又会调用它的子View的`measure`方法，让它的子View进行自我测量，然后根据它们自己测量的尺寸计算它们的位置，并把值保存下来，再根据这些值计算和保存自己的位置
ViewGroup继承子View，是一个抽象类，内部提供三个方法用于测量子控件：`measureChildren`，`measureChild`，`measureChildWithMargins`。但阅读源码发现ViewGroup并未
重写onMeasure方法，这是由于不同容器摆放位置不同，比如LinearLayout和RelativeLayout，这将导致测量的方式会有差异。如果我们自定义ViewGroup那就必须重写onMeasure方法测量
子控件的尺寸。

`measureChildWithMargins`和`measureChild`的区别就是父控件支不支持margin属性。在ViewGroup中有两个内部类：`LayoutParams`和`MarginLayoutParams`(后者继承自前者)，
这两个内部类就是ViewGroup的布局参数类。

自定义ViewGroup的Measure步骤基本如下

1.遍历测量子控件

2.合并子控件的尺寸，最终得到该自定义ViewGroup的测量值

3.调用`setMeasureDimension`方法存储测量值
