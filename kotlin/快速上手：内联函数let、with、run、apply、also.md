## Kotlin快速上手：内联函数let、with、run、apply、also

### let

常用场景：

对某一`nullable`变量统一处理

最后一行为返回值

```kotlin
val mObj = setUser()
mObj?.let{
    //使用it代替对象访问public属性和方法
    it.xxxxxx()
}
val i = "abc"?.let{
    println(it)
    it.length
}
//i最终结果为	3
```

### with

常用场景：

调用某对象的属性/方法时不用写对象.属性/方法

```kotlin
class User{
    var name
    var age
}

fun test(){
    val user = User()
    with (user){
        println(name)
        println(age)
    }
}
```

### run

常用场景：

适合`let`和`with`的任何场景，相当于两个结合

```kotlin
fun test(index: Int){
    val user = userList[index]
    user?.run{
        println(name)
        println(age)
    }
}
```

### apply

常用场景：

创建对象时给对象属性赋初始值，最终会返回该对象

```kotlin
val user = User().apply{
    name = "Lanier"
    age = 21
}
```

### also

常用场景：

适合`let`的任何场景，与其不同的是会返回对象本身而不是最后一行值

```kotlin
val i = "abc"?.let{
    println(it)
    it.length
}
//i最终结果为	abc
```

