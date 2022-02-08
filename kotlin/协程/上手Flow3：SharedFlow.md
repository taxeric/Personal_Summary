热流之一
## 结论
- SharedFlow无需提供初始值
- SharedFlow可以保留历史值

```kotlin
class SharedFlowTest {

    val sharedFlow = MutableSharedFlow<Int>()

    suspend fun emit(){
        val random = Random.nextInt(100)
        println("will emit -> $random")
        sharedFlow.emit(random)
    }
}

fun main() = runBlocking {
    val sharedFlowTest = SharedFlowTest()
    println("-----------------------------")

    val job1 = launch {
        sharedFlowTest.sharedFlow.collect {
            println("job1 collect -> $it")
        }
    }
    val job2 = launch {
        sharedFlowTest.sharedFlow.collect {
            println("job2 collect -> $it")
        }
    }
    repeat(5){
        delay(100)
        sharedFlowTest.emit()
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
will emit -> 80
job1 collect -> 80
job2 collect -> 80
will emit -> 71
job1 collect -> 71
job2 collect -> 71
will emit -> 82
job1 collect -> 82
job2 collect -> 82
will emit -> 26
job1 collect -> 26
job2 collect -> 26
will emit -> 39
job1 collect -> 39
job2 collect -> 39
-----------------------------
```
## MutableSharedFlow可选构造参数
- replay：当有新的订阅者时返回几个旧数据给他，返回数量为replay - 1，即replay = 3时返回两个旧数据。修改如下
```kotlin
class SharedFlowTest {

    val sharedFlow = MutableSharedFlow<Int>(replay = 3)

    suspend fun emit(){
        val random = Random.nextInt(100)
        println("will emit -> $random")
        sharedFlow.emit(random)
    }
}

fun main() = runBlocking {
    ...
    val job2 = launch {
        delay(1000) //延时1秒，模拟新的订阅者
        sharedFlowTest.sharedFlow.collect {
            println("job2 collect -> $it")
        }
    }
    repeat(5){
        delay(100)
        sharedFlowTest.emit()
    }
    delay(2000) //改为延时2秒
    ...
}
```
输出为
```
-----------------------------
will emit -> 17
job1 collect -> 17
will emit -> 62
job1 collect -> 62
will emit -> 71
job1 collect -> 71
will emit -> 14
job1 collect -> 14
will emit -> 63
job1 collect -> 63
job2 collect -> 71
job2 collect -> 14
job2 collect -> 63
-----------------------------
```
- extraBufferCapacity：除了replay之外缓冲区内的数量。当缓冲区还有空间时，emit不会挂起
- onBufferOverflow：缓冲区满后的处理策略，可选三种`BufferOverflow.SUSPEND`、`BufferOverflow.DROP_OLDEST`、`BufferOverflow.DROP_LATEST`
- - SUSPEND：被订阅者挂起
- - DROP_OLDEST：废弃缓冲区最旧的数据
- - DROP_LATEST：废弃缓冲区最新的数据

```kotlin
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T> {
    require(replay >= 0) { "replay cannot be negative, but was $replay" }
    require(extraBufferCapacity >= 0) { "extraBufferCapacity cannot be negative, but was $extraBufferCapacity" }
    require(replay > 0 || extraBufferCapacity > 0 || onBufferOverflow == BufferOverflow.SUSPEND) {
        "replay or extraBufferCapacity must be positive with non-default onBufferOverflow strategy $onBufferOverflow"
    }
    val bufferCapacity0 = replay + extraBufferCapacity
    val bufferCapacity = if (bufferCapacity0 < 0) Int.MAX_VALUE else bufferCapacity0 // coerce to MAX_VALUE on overflow
    return SharedFlowImpl(replay, bufferCapacity, onBufferOverflow)
}
```
从构造方法可以看出，缓冲区大小0 = 可返回的旧值数量 + 额外的缓冲区大小，若缓冲区大小0 < 0，则最终实现的缓冲区大小为Int.MAX_VALUE  

发射数据有两种方式：emit和tryEmit。当`MutableSharedFlow`中缓存数据量超过阈值时，`emit`方法和`tryEmit`方法的处理方式会有不同：  

- emit：当缓存策略为 BufferOverflow.SUSPEND 时，emit 方法会挂起，直到有新的缓存空间。
- tryEmit：tryEmit 会返回一个 Boolean 值，true 代表传递成功，false 代表会产生一个回调，让这次数据发射挂起，直到有新的缓存空间。

作者：九心
链接：https://juejin.cn/post/6937138168474894343

