在加载Bitmap对象时可以手动设置**inSampleSize**对图片进行缩放，顾名思义，缩放就是对图片宽高进行修改，进而减小所占内存。
# 创建时缩放
当创建Bitmap对象时，会有一个可选的Options对象
```java
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inSampleSize = 2;
    Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.test_pic, options);
```
上述代码表示宽高缩减为原来的1/2。**注意：inSampleSize的值必须大于1，且只能是2的整数倍**
# 从Drawable目录加载自动缩放
手机dpi与drawable目录对应关系
|DPI|分辨率|系统DPI|基准比例|对应目录|
|--|--|--|--|--|
|ldpi|240*320|120|0.75|ldpi（低密度）|
|mdpi|320*480|160|1|mdpi（中等密度）|
|hdpi|480*800|240|1.5|hdpi（高密度）|
|xdpi|720*1280|320|2|xdpi（超高密度）|
|xxdpi|1080*1920|480|3|xxdpi（超超高密度）|
|xxxdpi|2160*3840|640|4|xxxdpi（超超超高密度）|
### drawable目录选择流程
1. 如果手机是中等分辨率，则android会选择mdpi下的图片，如果有的话会被优先使用，且不会缩放
2. 如果mdpi没有相应图片，则会从更高一级的hdpi找，如果找到就进行**缩小处理**
3. 如果hdpi没有相应图片，则依次寻找
4. 如果更高密目的目录都没有，则向更低密度的ldpi找，如果找到就进行**放大处理**
5. 如果都没有，则会在默认的drawable目录找

**注意**

如果图片本身就比较大，又放到密度低的目录，则加载时候会导致占用内存变得非常大，引发OOM
此外，还有drawable-nodpi的目录，如果图片在该目录被找到，则无论设备dpi是多少，保留原图大小，不缩放
