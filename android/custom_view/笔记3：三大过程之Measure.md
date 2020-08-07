在布局中，每个View的大小不仅取决于自身，也受父控件影响。`onMeasure`方法作用是测量View的大小及模式，是由父控件计算子View视图大小时调用的。所有的父控件
都是ViewGroup的子类(例如LinearLayout)。

# View的Measure
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
这里的两个参数：widthMeasureSpec、heightMeasureSpec，指的是`父控件传递给当前子控件的一个建议值，即想把当前子控件宽高设为该参数值`，**不是**父控件的宽高**也不是**子控件的宽高

`setMeasuredDimension`方法用于保存子控件的宽度和高度，此方法必须被调用在`onMeasure`中。

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

从getDefaultSize方法可以看出，specSize值来自MeasureSpec.getSize(measureSpec)，故此问题在于子控件的specSize值是多少

## Answer
这个问题就要看看ViewGroup类了。在ViewGroup类中，有一个`getChildMeasureSpec`方法，该方法用于获取子控件的MeasureSpec值，简要代码如下
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
从这个方法中可以得知：子控件的specMode和specSize值不仅由自身的LayoutParams决定，也由父控件的MeasureSpec决定。在笔记1中，父控件(容器)为LinearLayout，其布局参数为match_parent。至此，就能解释为什么当子控件布局为warp_content时，最终显示与match_parent效果一致了。最后将该逻辑总结如下

![image](https://img-blog.csdnimg.cn/20200727143647717.png)

下面的是别人的总结

![image](https://upload-images.jianshu.io/upload_images/944365-6088d2d291bbae09.png?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)

一般情况下，控件的建议尺寸和实际尺寸一致

# ViewGroup的Measure
ViewGroup继承子View，是一个抽象类，内部提供三个方法用于测量子控件：`measureChildren`，`measureChild`，`measureChildWithMargins`。但阅读源码发现ViewGroup并未
重写onMeasure方法，这是由于不同容器摆放位置不同，比如LinearLayout和RelativeLayout，这将导致测量的方式会有差异。如果我们自定义ViewGroup那就必须重写onMeasure方法测量
子控件的尺寸。

`measureChildWithMargins`和`measureChild`的区别就是父控件支不支持margin属性。在ViewGroup中有两个内部类：`LayoutParams`和`MarginLayoutParams`(后者继承自前者)，
这两个内部类就是ViewGroup的布局参数类。

自定义ViewGroup的Measure步骤基本如下

1.遍历测量子控件

2.合并子控件的尺寸，最终得到该自定义ViewGroup的测量值

3.调用`setMeasureDimension`方法存储测量值
