在布局中，每个View的大小不仅取决于自身，也受父ViewGroup影响。`onMeasure`方法作用是测量View的大小及模式，是在父ViewGroup计算子View视图大小时被调用的。

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
这里的两个参数：widthMeasureSpec、heightMeasureSpec，指的是`父ViewGroup传递给当前子View的一个建议值，即想把当前子View宽高设为该参数值`，**不是**父ViewGroup的宽高**也不是**子View的宽高

`setMeasuredDimension`方法用于设置View的宽度和高度，此方法必须被调用在`onMeasure`中。

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
为什么子View布局设置warp_content也是填充父容器？

从getDefaultSize方法可以看出，specSize值来自MeasureSpec.getSize(measureSpec)，故此问题在于子View的specSize值是多少

## Answer
由笔记2知道，子View的MeasureSpec值是根据子View的布局参数（LayoutParams）和父容器的MeasureSpec值计算得来，在笔记1中，父容器为LinearLayout，布局参数为match_parent；
LinearLayout属于ViewGroup，ViewGroup类中有`getChildMeasureSpec`方法，该方法用于获取子View的MeasureSpec值，简要代码如下
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
到此，就能解释为什么当子View布局为warp_content时，最终显示与match_parent效果一致了。最后将上述逻辑总结如下

![image](https://img-blog.csdnimg.cn/20200727143647717.png)

下面的是别人的总结
![image](https://upload-images.jianshu.io/upload_images/944365-6088d2d291bbae09.png?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)

