在笔记1中提到了测量模式这个东西，现在就具体说一下....
# 三种测量模式
在View类中可以看到测量模式共有3种
|模式|值|描述|
|--|--|--|
|UNSPECIFIED|0|子View可以是任意大小，即父View无限制|
|EXACTLY|1|父View明确指定了子View的大小|
|AT_MOST|2|子View最大上限为父View大小|

**说明**
- 如果是UNSPECIFIED模式，则说明控件想要多大就要多大，用的比较少
- 如果是EXACTLY模式，则说明控件大小一般是确定值或match_parent
- 如果是AT_MOST模式，则说明控件大小一般是wrap_content

# 获取测量模式
Android已经提供了获取测量模式的方法，直接使用即可
- MeasureSpec.getSize(int measureSpec)
从提供的度量中提取大小
- MeasureSpec.getMode(int measureSpec)
从提供的测量中提取模式

在笔记1的第一个问题中，设置控件大小为`wrap_content`，此时测量出的模式就应该是`AT_MOST`，然后在`onMeasure`方法中进行判断：
- 如果宽度和高度测量模式都是`AT_MOST`，则给控件一个确定的值；
- 如果宽度(或高度)测量模式是`AT_MOST`，则给该宽度(或高度)一个确定的值，高度(或宽度)就设定为系统测量的值


OK，这篇就到这里
