经历了measure、layout后，终于到draw的环节。首先要明白一件事：先绘制的内容会被后绘制的覆盖掉

ViewGroup没有重写draw方法，因此所有的View都是调用子View的draw方法。但是具体是怎么draw ？在draw方法的注释中，已经给我们提示了：
```java
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
                   画背景
         *      2. If necessary, save the canvas' layers to prepare for fading
                   有必要的话，保存图层
         *      3. Draw view's content
                   绘制自个
         *      4. Draw children
                   如果有子View就绘制
         *      5. If necessary, draw the fading edges and restore layers
                   如果需要，绘制View的边缘
         *      6. Draw decorations (scrollbars for instance)
                   绘制装饰（如滚动条）
         */
```
第二步和第五步不是必须的，所以着重看其他四步就行

# 画背景
```java
    private void drawBackground(Canvas canvas) {
        final Drawable background = mBackground;
        if (background == null) {
            return;
        }
        //设置背景的边界
        setBackgroundBounds();

        ...

        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        if ((scrollX | scrollY) == 0) {
            background.draw(canvas);
        } else {
            //如果scrollX和scrollY有值，则对canvas的坐标进行偏移
            canvas.translate(scrollX, scrollY);
            //然后用drawable画背景
            background.draw(canvas);
            //最后复原
            canvas.translate(-scrollX, -scrollY);
        }
    }
```
# 画自个
调用`onDraw`方法，但由于自身的内容各不相同，所以是空方法
```java
    protected void onDraw(Canvas canvas) {
    }
````
# 画子View
```java
    protected void dispatchDraw(Canvas canvas) {
    }
```
不出意外，也是空实现。你可能想看看ViewGroup是怎么写的.....好吧，大部分都看不太懂，但是咱能看到一个叫`drawChild`的方法。实现如下：
```java
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```
look，最终也是调用child的draw方法。但是你也会发现，这里的draw和View的draw方法不一样，其实绘制的方法不是只有一个，其他方法在下没有找到，尴尬....不过**画什么会交给子View自己决定**这个是确定的

在笔记1中，我们直接在onDraw方法中画了一个圆，而且把绘制圆的代码写到onDraw后面，不过其实在上面或者下面都无所谓，删了onDraw也行。在别的自定义绘制中，更常见的是继承某种功能的控件，然后重写onDraw方法。举个例子：
```java
public class CustomView extends AppCompatImageView {
    ...

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
    }
}
```
这里我们继承自ImageView，然后写一种最常见的情况：为ImageView添加点缀
```java
public class CustomView extends AppCompatImageView {
    ...

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Paint paint = new Paint();
        paint.setColor(Color.YELLOW);
        paint.setStyle(Paint.Style.STROKE);
        paint.setStrokeWidth(3f);
        paint.setTextSize(30f);
        canvas.drawText("夏目友人帐", 0, 30, paint);
    }
}
```
效果如图

![添加文字](https://img-blog.csdnimg.cn/20200818212740241.png

那你有没有想过如果把这些代码写到super.onDraw上面会怎么样？

OK，这篇就到这里
