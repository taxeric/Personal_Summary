结论先行  
1. T.()->Unit 的函数体中可以直接使用T代表的对象，即用this代表对象
2. (T) -> Unit 将T表示的对象作为实参通过函数参数传递进来，供函数体使用，只相对于传了个T类型的参数
3. ()->Unit与T表示的对象没有直接联系，只能通过外部T实例的变量来访问对象
4. (T) -> Unit与 () -> Unit只是一个普通的函数，一个带有参数，类型为T，另一个没有参数，前者在函数体里用it访问
5. T.() -> Unit相当于扩展函数，在函数体内表示T对象，用this访问，而() -> Unit只传入函数，this表示当前类对象上下文

eg1
```kotlin
fun main() {
    launchx({
        "1  ->  haha"
    })
}

fun launchx(
	block: () -> String
){
    println(block())
}

output
1  ->  haha
```
eg2
```kotlin
fun main() {
    launchx2<String>({
        "2  ->  tax"
    })
    launchx2<Int>({
        999
    })
}

fun <T> launchx2(
	block: () -> T
){
    if (block() is String){
        println(block())
    } else {
        println("no str")
    }
}

output
2  ->  tax
no str
```

eg3
```kotlin
fun main(){
    launchx3("taxeric", {
        println(this)
    })
}

fun launchx3(
    str: String,
	block: String.() -> Unit
){
    block(str)
}

output
taxeric
```

eg4
```kotlin
fun main() {
    launchx3("taxeric", {
        println(this)
        566
    })
}

fun launchx3(
    str: String,
	block: String.() -> Int
){
    println(str.block() + 1)
}

output
taxeric
567
```

eg
```kotlin
fun main() {
    launchx4("taxeric"){
        println(it)
    }
}

fun launchx4(
    str: String,
	block: (String) -> Unit
){
    block(str)
}

output
taxeric
```

eg
```kotlin
data class Tx(val str: String = "")
fun main() {
    launchx5({
        "niu bi"
    }, {
        Tx(str = this)
    })
}

fun <T> launchx5(
    str: () -> String,
	block: String.() -> T
){
    val value = str().block()
    if (value is String){
        println("the value's type is String. -> $value")
    } else {
        println("no str. -> ${value.toString()}")
    }
}

output
no str. -> Tx(str=niu bi)
```
