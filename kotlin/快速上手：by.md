# by

关键字，举例`属性委托`

```kotlin
val a: String by Delegate()

class Delegate{
	operator fun getValue(thisRef: Any?, property: KProperty<*>): String{
        return "$thisRef ${property.name}"
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String){
        println("$value")
    }
}
```

当对变量`a`进行读写操作时，具体的实现是由Delegate来完成的，例打印a的值时，执行逻辑是由`Delegate`内的`getValue`完成的，同理给a设置值

>阿伟买日版游戏机，不用专门去一次日本，只需要通过代理商购买，即具体的购买操作由代理商完成，阿伟只是发起购买的操作

