## 区别
```kotlin
fun main() {
    println("Hello World")
    for (i in 0 until 5){
        println("until $i")
    }
    for (i in 0..5){
        println("..  $i")
    }
}
```
输出
```kotlin
Hello World
until 0
until 1
until 2
until 3
until 4
..  0
..  1
..  2
..  3
..  4
..  5
```

