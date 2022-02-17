一般用在泛型, 规则如下

out: 协变, 兼容自己及子类, 对应Java的 ? extends xxx  
in: 逆协变, 兼容自己及父类, 对应Java的 ? super xxx

example
```kotlin
abstract class A
abstract class B

class A1: A()
class B1: B()

class PermissionLauncher<out A, out B>//限定类型为A和B的子类型(包括A、B)

使用为
val p = PermissionLauncher<A, B>()//编译通过
val p1 = PermissionLauncher<A1, B1>()//编译通过
```

