理论上可以用NestedScrollView替代ScrollView，但会引发其他问题：
- RecyclerView复用机制失效  这是最主要的问题
- 界面初始位置不正确

对于一般的界面：顶部随意，底部RecyclerView，且顶部没有占满全屏，可以使用`CoordinatorLayout`，详情如下
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/app_bar_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <androidx.viewpager.widget.ViewPager
            android:id="@+id/home_vp"
            app:layout_scrollFlags="scroll"
            android:layout_width="match_parent"
            android:layout_height="220dp"/>
    </com.google.android.material.appbar.AppBarLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/home_rv"
        android:scrollbars="none"
        app:layout_behavior=".base.ui.ScrollRvBehavior"
        android:overScrollMode="never"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

ScrollRvBehavior
```java
class ScrollRvBehavior constructor(
    context: Context,
    attributeSet: AttributeSet
): CoordinatorLayout.Behavior<RecyclerView>(context, attributeSet) {

    override fun layoutDependsOn(
        parent: CoordinatorLayout,
        child: RecyclerView,
        dependency: View
    ): Boolean {
        return dependency is AppBarLayout || super.layoutDependsOn(parent, child, dependency)
    }

    override fun onDependentViewChanged(
        parent: CoordinatorLayout,
        child: RecyclerView,
        dependency: View
    ): Boolean {
        val offset = dependency.bottom - child.top
        child.translationY = offset.toFloat()
        return false
    }
}
```

其他的等想到再写吧


