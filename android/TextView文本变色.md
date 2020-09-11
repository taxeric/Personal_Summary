Shader称为着色器, 用于给图像着色
`LinearGradient`为Shader的子类, 用于实现线性渐变效果, 常用构造方法如下

```java
    /**
     * Create a shader that draws a linear gradient along a line.
     *
     * @param x0       渐变开始x坐标
     * @param y0       渐变开始y坐标
     * @param x1       渐变结束x坐标
     * @param y1       渐变结束y坐标
     * @param color0   起始位置的颜色
     * @param color1   结束位置的颜色
     * @param tile     着色器填充模式
     */
    public LinearGradient(float x0, float y0, float x1, float y1,
            @ColorInt int color0, @ColorInt int color1,
            @NonNull TileMode tile) {
        this(x0, y0, x1, y1, Color.pack(color0), Color.pack(color1), tile);
    }
```
模式有3种

```java
    public enum TileMode {
        /**
         * 边缘拉伸。使用边缘颜色对区域外的范围进行填充
         */
        CLAMP   (0),
        /**
         * 重复模式。在水平和垂直两个方向上重复填充
         */
        REPEAT  (1),
        /**
         * 镜像模式。在水平和垂直两个方向上以镜像的方式重复填充，相邻图像间有间隙
         */
        MIRROR  (2);
    }
```

实现变色效果

```java
public class TextViewColor extends androidx.appcompat.widget.AppCompatTextView {
	...
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        if (changed){
        	//实现的是从上往下渐变
            getPaint().setShader(new LinearGradient(0, 0,
                    0, getHeight(),
                    Color.RED, Color.BLUE,
                    Shader.TileMode.CLAMP));
        }
    }
}
```
也可以在textView中直接设置Shader

```java
	Shader shader = new LinearGradient(0, 0, 0, textView.getLineHeight(),
	        Color.GREEN, Color.BLUE, Shader.TileMode.REPEAT);
	textView.getPaint().setShader(shader);

```
测试布局

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">
	...

    <com.eric.specialeffects.TextViewColor
        android:text="北加尔.北境之地"
        android:gravity="center"
        android:padding="10dp"
        android:textSize="18sp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

    <TextView
        android:id="@+id/effect_text"
        android:text="北加尔.北境之地"
        android:gravity="center"
        android:padding="10dp"
        android:textSize="18sp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
</LinearLayout>
```
效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200511110427248.png)
