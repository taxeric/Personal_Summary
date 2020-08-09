经历了`measure`后，系统确定了View的大小，接下来就进入到Layout的过程。在该过程会确定控件的显示位置，即子控件在父控件的位置。与`measure`一致，该过程也有一个`onLayout`方法，且`layout`也是起到一个调度的作用，但与measure不同的是，layout方法会保存传进来的尺寸和位置（测量阶段是父View统一保存所有子View的布局，而布局阶段是子View自个保存自己的布局）。

在View类中，layout并未做实现，因为它没有子View，而对于ViewGroup，需要递归布局所有的子View

