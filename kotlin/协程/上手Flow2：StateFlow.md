热流之一，始终处于活跃状态且发射的值处于内存中，只有垃圾回收根未涉及其他引用才被回收，需要指定初始值

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
    repeat(5){
        stateFlowTest.emit()
    }
    val job = launch {
        stateFlowTest.mutableStateFlow.collect {
            println("collect -> $it")
        }
    }
    delay(100)
    job.cancel()
    println("-----------------------------")
}
```
输出为
```
-----------------------------
will emit -> 77
will emit -> 79
will emit -> 42
will emit -> 21
will emit -> 23
collect -> 23
-----------------------------
```
修改如下
```kotlin
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


