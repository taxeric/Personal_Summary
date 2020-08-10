看别的博文了解到整个自定义View包括两大过程：布局和绘制，而对应的有有三个方法：measure、layout、draw，分别表示测量、布局、绘制。

# 布局
布局过程就是把界面上的所有控件，按其大小摆放在正确的位置。一般情况下咱不用管这个过程，因为自带控件的布局过程都已经写好了。我初学Android基本没有意识到这个过程的存在，就是我写了xml，然后他显示，哈哈。
所谓布局过程就包含了`测量`和`布局`，界面从根布局向下递归测量出每一级，每一个子View的尺寸和位置，将这些值赋给子View，然后....这就完成了布局过程

## 自定义布局过程
具体来说有三种方式：
- 重写onMeasure修改已有View的尺寸
- 重写onMeasure重新自定义View的尺寸
- 重写onMeasure和onLayout重新自定义ViewGroup的内部布局

### 重写onMeasure修改已有尺寸
1. 重写onMeasure，调用super.onMeasure()
2. 用getMeasureWidth()和getMeasureHeight()取得之前测得的尺寸，并利用该值计算新的最终尺寸
3. 用setMeasureDimension()保存最终尺寸

### 重写onMeasure重新自定义View的尺寸
1. 重写onMeasure，自己算尺寸
2. 将计算结果用resolveSize()过滤一次（这个方法主要作用就是做对比，看看你算的结果是否在限制尺寸内，具体看源码）
3. 用setMeasureDimension()保存最终尺寸

### 重写onMeasure和onLayout重新自定义ViewGroup的内部布局
1. 重写onMeasure计算内部布局
2. 重写onLayout拜访子View

# 绘制
即`draw`方法。在布局过程中，每个子View都存储了自己的尺寸和位置，接下来就按这些属性绘制自身即可



OK，这篇就到这里
