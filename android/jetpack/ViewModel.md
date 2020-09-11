ViewModel是专门用于存放和管理数据的类, 即与界面相关的数据都应放在ViewModel中. 此外, ViewModel有一个非常重要的特性: 当Activity被意外重新创建(屏幕旋转等)时, ViewModel不会被重新创建, 故界面上的数据不会被丢失, 只有当Activity被销毁时才会一起销毁.

## 简单使用
导入扩展依赖
```java
    //使用ViewModel扩展组件
    implementation 'androidx.lifecycle:lifecycle-extensions:2.2.0'
```
自定义MainViewModel类继承自ViewModel

```java
open class MainViewModel: ViewModel() {
    
    private val name = "abc"
    private val age = 20

    fun getName(): String = name

    fun getAge(): Int = age
}
```
Activity使用

```java
class MainActivity : AppCompatActivity() {

    private lateinit var mainViewModel: MainViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mainViewModel = ViewModelProvider(this)[MainViewModel::class.java]
        LogUtils.i(mainViewModel.getName()+"  "+mainViewModel.getAge())
    }
}
```
注意：ViewModel绝对不能引用视图、生命周期或任何可能包含对活动上下文的引用的类
