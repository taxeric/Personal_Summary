经历了measure、layout后，终于到draw的环节。首先要明白一件事：先绘制的内容会被后绘制的覆盖掉

ViewGroup没有重写draw方法，因此所有的View都是调用子View的draw方法。但是具体是怎么draw ？在draw方法的注释中，已经给我们提示了：
```java
        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
                   绘制背景
         *      2. If necessary, save the canvas' layers to prepare for fading
                   有必要的话，保存图层
         *      3. Draw view's content
                   绘制自个
         *      4. Draw children
                   如果有子View就绘制
         *      5. If necessary, draw the fading edges and restore layers
                   如果需要，绘制View的边缘
         *      6. Draw decorations (scrollbars for instance)
                   绘制前景，装饰（如滚动条）
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

那有没有想过如果把这些代码写到super.onDraw上面会怎么样？常见的一个场景就是强调色，按照上面的逻辑，会先绘制强调色，然后绘制文本：
```java
public class CustomView extends AppCompatTextView {
    ...

    @Override
    protected void onDraw(Canvas canvas) {
        Layout layout = getLayout();
        Paint paint = new Paint();
        paint.setColor(Color.YELLOW);
        RectF bounds = new RectF();
        bounds.left = layout.getLineLeft(0);
        bounds.right = layout.getLineRight(0);
        bounds.top = layout.getLineTop(0);
        bounds.bottom = layout.getLineBottom(0);
        canvas.drawRect(bounds, paint);
        super.onDraw(canvas);
    }
}
```

当我们需要在ViewGroup中绘制，比如LinearLayout时，如果将绘制的代码写到super.onDraw()下面，按照上面的逻辑，会先绘制写的代码，然后绘制子View，这并不是我们想要的。怎样才能让LinearLayout的绘制内容在子View上面？只需将绘制的代码写到子View绘制完成后就🉑：
```java
public class CustomView extends LinearLayout {
    ...

    @Override
    protected void dispatchDraw(Canvas canvas) {
        super.dispatchDraw(canvas);
        Paint paint = new Paint();
        paint.setColor(Color.BLUE);
        paint.setStyle(Paint.Style.FILL);
        canvas.drawCircle(50, 50, 50, paint);
        paint.setColor(Color.GREEN);
        canvas.drawCircle(180, 200, 80, paint);
    }
}
```
而将绘制的代码写到super.dispatchDraw()上面，效果和写到super.onDraw()是一样的

# 画前景
```java
    public void onDrawForeground(Canvas canvas) {
        onDrawScrollIndicators(canvas);
        onDrawScrollBars(canvas);

        final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
        if (foreground != null) {
            if (mForegroundInfo.mBoundsChanged) {
                mForegroundInfo.mBoundsChanged = false;
                final Rect selfBounds = mForegroundInfo.mSelfBounds;
                final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

                if (mForegroundInfo.mInsidePadding) {
                    selfBounds.set(0, 0, getWidth(), getHeight());
                } else {
                    selfBounds.set(getPaddingLeft(), getPaddingTop(),
                            getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
                }

                final int ld = getLayoutDirection();
                Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                        foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
                foreground.setBounds(overlayBounds);
            }

            foreground.draw(canvas);
        }
    }
```
该方法是API23时引入的，会依次绘制滑动边缘渐变、滑动条和前景。
```java
public class CustomView extends LinearLayout {
    ...

    @Override
    public void onDrawForeground(Canvas canvas) {
        //绘制内容在绘制前景之前
        super.onDrawForeground(canvas);
        //绘制内容在绘制前景之后
    }
}
```
如果我们将绘制代码写到super.onDrawForeground()上面，那么我们绘制的内容就会被前景遮盖，写到下面会覆盖前景。

![总结](http://wx3.sinaimg.cn/large/006tKfTcly1fii5jk7l19j30q70e0di5.jpg)

# 注意
以下摘自Hencoder

1. 出于效率的考虑，ViewGroup 默认会绕过 draw() 方法，换而直接执行 dispatchDraw()，以此来简化绘制流程。所以如果你自定义了某个 ViewGroup 的子类（比如 LinearLayout）并且需要在它的除 dispatchDraw() 以外的任何一个绘制方法内绘制内容，你可能会需要调用 View.setWillNotDraw(false) 这行代码来切换到完整的绘制流程（是「可能」而不是「必须」的原因是，有些 ViewGroup 是已经调用过  setWillNotDraw(false) 了的，例如 ScrollView）。
2. 有的时候，一段绘制代码写在不同的绘制方法中效果是一样的，这时你可以选一个自己喜欢或者习惯的绘制方法来重写。但有一个例外：如果绘制代码既可以写在 onDraw() 里，也可以写在其他绘制方法里，那么优先写在 onDraw() ，因为 Android 有相关的优化，可以在不需要重绘的时候自动跳过  onDraw() 的重复执行，以提升开发效率。享受这种优化的只有 onDraw() 一个方法。

OK，这篇就到这里
