在笔记1中提到了测量模式这个东西，现在就具体说一下....

# MeasureSpec
测量模式在Android中的表现是MeasureSpec类，该类也可以理解为View的测量规则。一般来说，子控件的宽高不会超出父控件，但是父控件需要测量子控件的宽高，以便设置自己的宽高。在测量时，会传给子控件两个int类型的信息，就是在笔记1中`onMeasure`方法的两个参数
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
```
每个参数的值都包含两个信息：**SpecMode、SpecSize**，我们知道int代表的是一个4字节32位的数据，在这两个参数中，前面高2位是SpecMode，后低30位是SpecSize

# SpecMode的三种取值
在MeasureSpec类中可以看到测量模式共有3种
|模式|值|描述|
|--|--|--|
|UNSPECIFIED|0|子View可以是任意大小，即父View无限制|
|EXACTLY|1|父View明确指定了子View的大小|
|AT_MOST|2|子View最大上限为父View大小|

**说明**
- 如果是UNSPECIFIED模式，则说明控件想要多大就要多大，用的比较少，父控件不对子控件做任何限制，常见于ListView、ScrollView
- 如果是EXACTLY模式，则说明控件大小一般是确定值或match_parent，父控件给子控件明确指定大小
- 如果是AT_MOST模式，则说明控件大小一般是wrap_content，父控件给子控件指定最大参考尺寸，**希望**子控件不要超过这个尺寸

# 获取测量模式
Android已经提供了获取测量模式的方法，直接使用即可
- MeasureSpec.getSize(int measureSpec)
从提供的度量中提取大小
- MeasureSpec.getMode(int measureSpec)
从提供的测量中提取模式

在笔记1的第一个问题中，设置控件大小为`wrap_content`，此时测量出的模式就应该是`AT_MOST`，然后在`onMeasure`方法中进行判断：
- 如果宽度和高度测量模式都是`AT_MOST`，则给控件一个确定的值；
- 如果宽度(或高度)测量模式是`AT_MOST`，则给该宽度(或高度)一个确定的值，高度(或宽度)就设定为系统测量的值

# 题外话：LayoutParams
除测量模式外，这里要提一个`LayoutParams`的概念。LayoutParams是ViewGroup的内部类，简单说就是布局参数，包含View的宽高等信息，每一个ViewGroup的子类都有对应的LayoutParams，比如`LinearLayout.LayoutParams、RelativeLayout.LayoutParams`等
|值|含义|
|--|--|
|LayoutParams.MATCH_PARENT|xml中设置View的属性为match_parent或fill_parent|
|LayoutParams.WRAP_CONTENT|xml中设置View的属性为wrap_content|


OK，这篇就到这里
