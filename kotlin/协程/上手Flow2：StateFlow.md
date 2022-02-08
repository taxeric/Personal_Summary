热流之一，始终处于活跃状态且发射的值处于内存中，只有垃圾回收根未涉及其他引用才被回收，需要指定初始值。（安卓开发中StateFlow与LiveData作用基本一致）  
## 结论
- StateFlow只会把最新值emit给订阅者，与订阅者数量无关
- StateFlow需要提供初始值  

```kotlin
class StateFlowTest {

    val mutableStateFlow = MutableStateFlow(0)

    suspend fun emit(){
        val random = Random.nextInt(100)
        println("will emit -> $random")
        mutableStateFlow.emit(random)
    }
}

fun main() = runBlocking {
    val stateFlowTest = StateFlowTest()
    println("-----------------------------")

    val job = launch {
        stateFlowTest.mutableStateFlow.collect {
            println("collect -> $it")
        }
    }
    repeat(5){
        delay(100)
        stateFlowTest.emit()
    }
    delay(1000)
    job.cancel()
    println("-----------------------------")
}
```
输出为
```
-----------------------------
collect -> 0
will emit -> 1
collect -> 1
will emit -> 18
collect -> 18
will emit -> 37
collect -> 37
will emit -> 74
collect -> 74
will emit -> 53
collect -> 53
-----------------------------
```
修改如下
```kotlin
fun main() = runBlocking {
    val stateFlowTest = StateFlowTest()
    println("-----------------------------")

    val job1 = launch {
        stateFlowTest.mutableStateFlow.collect {
            println("job1 collect -> $it")
        }
    }
    val job2 = launch {
        stateFlowTest.mutableStateFlow.collect {
            println("job2 collect -> $it")
        }
    }
    repeat(5){
        delay(100)
        stateFlowTest.emit()
    }
    delay(1000)
    job1.cancel()
    job2.cancel()
    println("-----------------------------")
}
```
输出为
```
-----------------------------
job1 collect -> 0
job2 collect -> 0
will emit -> 23
job1 collect -> 23
job2 collect -> 23
will emit -> 8
job1 collect -> 8
job2 collect -> 8
will emit -> 38
job1 collect -> 38
job2 collect -> 38
will emit -> 72
job1 collect -> 72
job2 collect -> 72
will emit -> 81
job1 collect -> 81
job2 collect -> 81
-----------------------------
```
**只要上游发生变化则发送给下游**

## 安卓开发可能会遇到的坑
### 1.未收集到值?
安卓开发会遇到当第二次collect收集的结果和上一次一致则没有事件产生，查阅源码为
```kotlin
    private fun updateState(expectedState: Any?, newState: Any): Boolean {
        var curSequence = 0
        var curSlots: Array<StateFlowSlot?>? = this.slots // benign race, we will not use it
        synchronized(this) {
            val oldState = _state.value
            if (expectedState != null && oldState != expectedState) return false // CAS support
            if (oldState == newState) return true // Don't do anything if value is not changing, but CAS -> true    <------------------
            ...
        }
        ...
    }
```
可得若新值和旧值一直则没有事件  
解决方式：可以新建类继承MutableStateFlow后重写`hashCode`和`equals`方法
### 2.页面进入后台后有其他地方调用了`emit`导致崩溃?
这是因为`collect`不会因为页面停止/销毁而自动停止接收新值  
解决方式1：在进入后台前手动取消订阅  
解决方式2：引入依赖`androidx.lifecycle:lifecycle-runtime-ktx:2.4.0`，在收集数据时使用`lifecycle.repeatOnLiftcycle(特定状态){}`。它会将当前协程的执行中断，直到特定事件发生。举例
```kotlin
        viewmodel.viewModelScope.launch {
            lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewmodel.lazyInitStateFlow.collect {
                    //当页面处于Started状态才收集数据
                }
            }
        }

```
**解决方式2待实践**
