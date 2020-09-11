LiveData是Jetpack提供的响应式编程组件, 支持泛型, 在数据发生变化时通知观察者, 一般与ViewModel结合使用
## 优点
- 能够保证数据和UI统一

- 减少内存泄漏

- 当Activity停止时不会引起崩溃

- 不需要额外的手动处理来响应生命周期的变化

- 组件和数据相关的内容能实时更新

- 针对configuration change时，不需要额外的处理来保存数据

- 资源共享

> 作者：斌林诚上  https://www.jianshu.com/p/6651046cd4d8

## 简单使用

```java
open class MainViewModel: ViewModel() {

 	...
    val data = MutableLiveData<String>()
    
    fun setData(){
        data.value = "Hello world"
    }

    fun getData(): String{
        if (TextUtils.isEmpty(data.value)){
            setData("Hello world")
        }
        return data.value!!
    }
}
```
在上一节自定义的ViewModel中新增data, 类型为`MutableLiveData`, 指定泛型为String. `MutableLiveData`是一种可变的LiveData, 主要有3中读写数据的方法.

- getValue()
- setValue()  
只能在主线程调用
- postValue()
用于在非主线程设置数据


对Activity作如下修改
```java
class MainActivity : AppCompatActivity() {

    private lateinit var mainViewModel: MainViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mainViewModel = ViewModelProvider(this)[MainViewModel::class.java]
        
        add_num.setOnClickListener {
            mainViewModel.setData(Random.nextInt(10).toString())
        }

        mainViewModel.data.observe(this, Observer {
            show_msg.text = mainViewModel.getData()
        })
    }
}
```
对于LiveData对象来说, 任何一个该对象都可以调用`observe`方法, 该方法用于监听数据的变化. `observe`需要两个参数: 第一个参数是`LifecycleOwner`对象, 对于Activity来说, 本身就是一个`LifecycleOwner`对象, 故直接传`this`即可, 第二个参数为`Observer`接口, 当MainViewModel中data的数据发生变化时, 就会回调到这里, 故只需将回调的结果显示到页面即可

相对的, LiveData还有一个`observeForever`方法, 该方法只需传递`Oberver`接口, 即不与生命周期挂钩, 只要数据发生改变就会回调该接口, 但是需要手动`removeObserver`	

这时有一个问题, 即把MainViewModel的data变量暴露给外部, 外部也可以通过data的value直接设置数据, 这并非我们想要的. 故可以对MainViewModel作如下修改

```java
open class MainViewModel: ViewModel() {

    private val data = MutableLiveData<String>()
    val result: LiveData<String>get() = data

    fun setData(msg: String){
        data.value = msg
    }

    fun getData(): String{
        if (TextUtils.isEmpty(data.value)){
            setData("Hello world")
        }
        return data.value!!
    }
}
```
把data加上private修饰符, 然后定义result变量, 其类型为不可变的LiveData, 将data作为该变量get方法的返回值, 这样外部调用result时返回的是data实例, 但无法给result.value设置数据

## 自定义LiveData
举个例子吧    此处的`NetworkHelper`是 [监听网络变化](https://blog.csdn.net/AneTist/article/details/105710507)
```java
class NetworkLiveData constructor(context: Context?): LiveData<Boolean>(),
    NetworkHelper.ChangeListener {

    private var helper: NetworkHelper = NetworkHelper()

    init {
        helper.registerListener(this, context!!)
    }

    //处于激活的状态
    override fun onActive() {
        super.onActive()
    }

    //处于销毁的状态
    override fun onInactive() {
        super.onInactive()
        helper.unregisterListener()
    }

    override fun onChange(isAvailable: Boolean) {
        postValue(isAvailable)
    }
}
```

使用  (在Activity中)

```java
        val i = MainLiveData(this)
        i.observe(this, Observer {
            LogUtils.i("当前网络变化: ${i.value}")
        })
```

这个NetworkLiveData用于网络监听, 可以改成单例形式, 随时监听
