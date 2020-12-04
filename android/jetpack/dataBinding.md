## 快速上手
模块的build.gradle添加(如果没有的话)：
```kotlin
    buildFeatures{
        dataBinding true
    }
```
等待构建完成

在布局里面光标移动到根布局，alt+enter，选择有`binding xxxxx`的，会自动生成
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">
        
        ....
    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```
写一个类
```kotlin
data class TModel(val name: String)
```

映射到布局
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <import type="com.eric.pokemon.TModel"
            alias="TestModel"/>
        <variable
            name="mModel"
            type="TestModel" />
    </data>
    ...
</layout>
```

参数说明
- import-type：引入数据类型
- import-alias：别名，防止不同包下类名重复
- variable-name：变量名
- variable-type：变量类型

**关于变量类型，如果import写了别名就用别名，否则用类名；没有写import则要写完整包名**

引用到TextView
```xml
        <TextView
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{mModel.name}"/>
```

做完上述操作后会创建一个<布局名Binding>的类，比如布局为`R.layout.activity_main`，则创建的类为`ActivityMainBinding`。如果没有，rebuild

在Activity中绑定视图
```kotlin
class MainActivity : AppCompatActivity() {

    private var binding = ActivityMainBinding()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        binding.mModel = TModel("Pokemon")
    }

    override fun onDestroy() {
        super.onDestroy()
        //解绑
        binding.unbind()
    }
}
```


