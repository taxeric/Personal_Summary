在上一节的笔记中提到了`onDraw`方法，其中它的参数`Canvas`字面意思就是 画布，而且在笔记1中也画了一个简单的圆。这一节就了解一些画布的方法

# 常用操作（来自GcsSloop）
|类型|API|说明|
|--|--|--|
|绘制颜色|drawColor，drawRGB，drawARGB|使用单一颜色填充整个画布|
|基本形状|drawPoint, drawPoints, drawLine, drawLines, drawRect, drawRoundRect, drawOval, drawCircle, drawArc|依次为 点、线、矩形、圆角矩形、椭圆、圆、圆弧|
|绘制图片|drawBitmap, drawPicture|绘制位图和图片|
|绘制文本|drawText, drawPosText, drawTextOnPath|依次为 绘制文字、绘制文字时指定每个文字位置、根据路径绘制文字|
|绘制路径|drawPath|绘制路径，绘制贝塞尔曲线时也需要用到该函数|
|顶点操作|drawVertices, drawBitmapMesh|通过对顶点操作可以使图像形变，drawVertices直接对画布作用、 drawBitmapMesh只对绘制的Bitmap作用|
|画布裁剪|clipPath, clipRect|设置画布的显示区域|
|画布快照|save, restore, saveLayerXxx, restoreToCount, getSaveCount|依次为 保存当前状态、 回滚到上一次保存的状态、 保存图层状态、 回滚到指定状态、 获取保存次数|
|画布变换|translate, scale, rotate, skew|依次为 位移、缩放、 旋转、错切|
|Matrix(矩阵)|getMatrix, setMatrix, concat|实际上画布的位移，缩放等操作的都是图像矩阵Matrix， 只不过Matrix比较难以理解和使用，故封装了一些常用的方法|

# 基本操作
## translate  位移
把坐标系移动，即**位移是基于当前位置移动，不是每次基于屏幕左上角移动**
```java
        // 绘制一个黑色圆形
        mPaint.setColor(Color.BLACK);
        canvas.translate(200,200);
        canvas.drawCircle(0,0,100,mPaint);

        // 绘制一个蓝色圆形
        mPaint.setColor(Color.BLUE);
        canvas.translate(200,200);
        canvas.drawCircle(0,0,100,mPaint);
```
## scale  缩放
有两个方法
```java
public void scale(float sx, float sy)
public final void scale(float sx, float sy, float px, float py)
```
前两个参数就是x轴和y轴的缩放比例，下面的方法可以控制缩放中心位置。缩放比例取值如下：
|取值|说明|
|--|--|
|(-∞, -1)|先根据缩放中心放大n倍，再根据中心轴进行翻转|
|-1|根据缩放中心轴进行翻转|
|(-1, 0)|先根据缩放中心缩小到n，再根据中心轴进行翻转|
|0|不会显示，若sx为0，则宽度为0，不会显示，sy同理|
|(0, 1)|根据缩放中心缩小到n|
|1|没有变化|
|(1, +∞)|根据缩放中心放大n倍|



