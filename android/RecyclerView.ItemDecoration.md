# 是什么
众所周知，RecyclerView本身是没有分割线的，对此，谷歌提供`ItemDecoration`类，意为项目装饰，用于实现分割线的效果。（而且还能实现各种🐂🍺又炫酷的样式）
# 怎么做
事实上，RecyclerView已经提供了一个参考的分割线样例，我们可以直接拿来用，也可以按照它的写法写自己的分割线样式
```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ...
        //该样式是已经写好的例子，直接用
        DividerItemDecoration d = new DividerItemDecoration(this, LinearLayout.VERTICAL);
        d.setDrawable(getResources().getDrawable(R.drawable.divider_item));
        recyclerView.addItemDecoration(d);
        ...
    }
```
这里的divider_item如下所示
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <!--指定填充颜色-->
    <solid android:color="#DF8961"/>
    <!--指定高度-->
    <size android:height="5dp"/>
</shape>
```
你可以写出更好的样式
PS：如果不指定样式，将会显示默认样式。
## 自定义
按照已有的样例代码，我们可以照猫画虎的写出自己想要的样子
```java
public class EItemDecoration extends RecyclerView.ItemDecoration {

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);
    }

    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
    }
}
```
看ItemDecoration类，发现有两个关于绘制的方法：
- onDraw()
- onDrawOver()
看注释，大概意思是**onDraw在item绘制之前绘制，在item视图的下面，而onDrawOver在item绘制之后绘制，在item视图的上面**

此外还有一个`getItemOffsets`方法，大概意思是**检索给定项的偏移量，即给定要绘制自定义View的空间**。不太理解什么意思，先看看这个回调在哪被调用的吧
```java
    Rect getItemDecorInsetsForChild(View child) {
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        ...
        final Rect insets = lp.mDecorInsets;
        insets.set(0, 0, 0, 0);
        final int decorCount = mItemDecorations.size();
        for (int i = 0; i < decorCount; i++) {
            mTempRect.set(0, 0, 0, 0);
            mItemDecorations.get(i).getItemOffsets(mTempRect, child, this, mState);
            insets.left += mTempRect.left;
            insets.top += mTempRect.top;
            insets.right += mTempRect.right;
            insets.bottom += mTempRect.bottom;
        }
        lp.mInsetsDirty = false;
        return insets;
    }
```
连个注释都没，看个🔨.....莫慌，理一下代码。在循环的上面，`final Rect insets = lp.mDecorInsets`，看看 mDecorInsets 是个啥
```java
final Rect mDecorInsets = new Rect();
```
好的，接下来返回那个方法，往下看，把insets的左上右下都设成0了；再往下，取 mItemDecorations 的大小，这个集合看的出来是存放啥的吧？看看哪里添加了数据：
```java
    public void addItemDecoration(@NonNull ItemDecoration decor, int index) {
        if (mLayout != null) {
            mLayout.assertNotInLayoutOrScroll("Cannot add item decoration during a scroll  or"
                    + " layout");
        }
        if (mItemDecorations.isEmpty()) {
            setWillNotDraw(false);
        }
        if (index < 0) {
            mItemDecorations.add(decor);
        } else {
            mItemDecorations.add(index, decor);
        }
        markItemDecorInsetsDirty();
        requestLayout();
    }
```
有点眼熟....这就是咱为RecyclerView设置分割线的方法啊，往上看看这个index传了多少
```java
    public void addItemDecoration(@NonNull ItemDecoration decor) {
        addItemDecoration(decor, -1);
    }
```
OK，默认传的-1，所以推测集合大小应该是1，即`getItemDecorInsetsForChild`方法里面for循环只循环一次。继续看循环里面干了啥，先把 mTempRect 的左上右下置为0，咱知道它就是个矩形对象就行，就不细看是怎么来的了，因为它在这里就只是置四个边距。再往下就调用了`getItemOffsets`方法了，然后把调用之后的对象里面的值累加到 insets 对应的值上，最后返回 insets。再看看这个方法是被谁调用的....发现`measureChild`里面调用了，就先看看这个
```java
        public void measureChild(@NonNull View child, int widthUsed, int heightUsed) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);
            widthUsed += insets.left + insets.right;
            heightUsed += insets.top + insets.bottom;
            final int widthSpec = getChildMeasureSpec(getWidth(), getWidthMode(),
                    getPaddingLeft() + getPaddingRight() + widthUsed, lp.width,
                    canScrollHorizontally());
            final int heightSpec = getChildMeasureSpec(getHeight(), getHeightMode(),
                    getPaddingTop() + getPaddingBottom() + heightUsed, lp.height,
                    canScrollVertically());
            if (shouldMeasureChild(child, widthSpec, heightSpec, lp)) {
                child.measure(widthSpec, heightSpec);
            }
        }
```
哦豁，这代码很眼熟啊，用到了`getChildMeasureSpec`，而且上一步得出来的 insets 值加到了子View的padding里面，所以到这里推测在`getItemOffsets`方法里面，咱可以通过设置`outRect`的左上右下属性改变子View的padding值。OK，来试试
```java
public class EItemDecoration extends RecyclerView.ItemDecoration {

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        //反正里面也是空实现，直接注释掉算了
        //super.getItemOffsets(outRect, view, parent, state);
        outRect.top = 10;
        outRect.left = 10;
        outRect.right = 10;
        outRect.bottom = 10;
    }
    ...
}
```
运行～  结果和猜测的一致。😆😆😆仔细想想，这样也算透明的分割线了😁

接下来照猫画虎的把DividerItemDecoration里面画Drawable的代码复制到自定义的里面
```java
public class EItemDecoration extends RecyclerView.ItemDecoration {

    private Drawable mDivider;

    public EItemDecoration(Drawable mDivider){
        this.mDivider = mDivider;
    }
    
    ...
    @Override
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        super.onDraw(c, parent, state);
        draw(c, parent);
    }

    private void draw(Canvas canvas, RecyclerView parent){
        canvas.save();
        Rect mBounds = new Rect();
        final int left;
        final int right;
        //noinspection AndroidLintNewApi - NewApi lint fails to handle overrides.
        if (parent.getClipToPadding()) {
            left = parent.getPaddingLeft();
            right = parent.getWidth() - parent.getPaddingRight();
            canvas.clipRect(left, parent.getPaddingTop(), right,
                    parent.getHeight() - parent.getPaddingBottom());
        } else {
            left = 0;
            right = parent.getWidth();
        }

        final int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = parent.getChildAt(i);
            parent.getDecoratedBoundsWithMargins(child, mBounds);
            final int bottom = mBounds.bottom + Math.round(child.getTranslationY());
            final int top = bottom - mDivider.getIntrinsicHeight();
            mDivider.setBounds(left, top, right, bottom);
            mDivider.draw(canvas);
        }
        canvas.restore();
    }
}
```
缺啥补啥，然后运行...emmm...底部padding显示貌似有点问题....看看例子是怎么写的。
```java
outRect.set(0, 0, 0, mDivider.getIntrinsicHeight());
```
算了，就照它的写吧....再运行，正常了。
（写这篇的时候测了很多种情况，我直接写个总结算了）
拿outRect的bottom来说，mDivider.getIntrinsicHeight()返回的值是drawable里面`size`标签`height`属性的px的值，也就是

height(dp) => height(px) = mDivider.getIntrinsicHeight(px) = outRect.bottom

就是你写分割线时，两个item之间的高度，单位是像素（px）。如果outRect.bottom = 0，则当item布局是透明时分割线从item的底部开始向上绘制，高度还是那个转换成px后高度

md，写不下去了，看看别人怎么写的
https://www.wecando.cc/article/9

OK，这篇就到这里
