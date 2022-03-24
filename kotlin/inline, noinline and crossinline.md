## inline
在编译时期将方法复制到调用的地方。举例
```kotlin
fun main() {
    var result = 0
    repeat(10){
        result += test(it)
    }
}

inline fun test(i: Int) = i + 2
```
编译成Java:
```java
   public static final void main() {
      int result = 0;
      byte var1 = 10;
      boolean var2 = false;
      boolean var3 = false;
      int var8 = 0;

      for(byte var4 = var1; var8 < var4; ++var8) {
         int var6 = false;
         int $i$f$test = false;
         result += var8 + 2;
      }

   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final int test(int i) {
      int $i$f$test = 0;
      return i + 2;
   }
```
删除`inline`关键字
```java
   public static final void main() {
      int result = 0;
      byte var1 = 10;
      boolean var2 = false;
      boolean var3 = false;
      int var7 = 0;

      for(byte var4 = var1; var7 < var4; ++var7) {
         int var6 = false;
         result += test(var7);
      }

   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final int test(int i) {
      return i + 2;
   }
```
可以看到不使用`inline`后最终还是调用方法。当inline修饰的`public`方法调用`private`修饰的变量/常量时会报错:
```kotlin
fun main() {
    var result = 0
    repeat(10){
        result += test(it)
    }
}

private val x = 100

inline fun test(i: Int) = i + x    //<----- 编译不通过
```
**注意**: 当被`inline`修饰的方法内部有`return`时，调用方也会return，因为是将方法[复制]到调用方

## noinline
当某些被`inline`修饰的方法不想使用`inline`的特性时可使用该关键字。举例
```kotlin
fun main() {
    test(10) {
        println("haha")
    }
}

inline fun test(i: Int, noinline method: () -> Unit){
    println("transport -> $i")
    method()
}
```
转成Java后:
```java
   public static final void main() {
      byte i$iv = 100;
      Function0 method$iv = (Function0)null.INSTANCE;    //<---- 这里使用的是jvm的接口，为匿名内部类形式。参见 [kotlin.jvm.functions]
      int $i$f$test = false;
      String var3 = "transport -> " + i$iv;
      boolean var4 = false;
      System.out.println(var3);
      method$iv.invoke();                                //<---- 调用上述接口的invoke方法
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void test(int i, @NotNull Function0 method) {
      int $i$f$test = 0;
      Intrinsics.checkParameterIsNotNull(method, "method");
      String var3 = "transport -> " + i;
      boolean var4 = false;
      System.out.println(var3);
      method.invoke();
   }
```
如果没有使用该关键字，则如下:
```java
   public static final void main() {
      int i$iv = 100;
      int $i$f$test = false;
      String var2 = "transport -> " + i$iv;
      boolean var3 = false;
      System.out.println(var2);
      int var4 = false;
      String var5 = "haha";
      boolean var6 = false;
      System.out.println(var5);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void test(int i, @NotNull Function0 method) {
      int $i$f$test = 0;
      Intrinsics.checkParameterIsNotNull(method, "method");
      String var3 = "transport -> " + i;
      boolean var4 = false;
      System.out.println(var3);
      method.invoke();
   }
```

## crossinline
与`inline`类似，当lambda方法被该关键字修饰时，return只能返回当前lambda方法。换言之，修饰过后不会影响调用方后续的流程
```kotlin
fun main() {
    test(100) {
        println("haha")
        return
    }
    println("xyz")
}

inline fun test(i: Int, method: () -> Unit){
    println("transport -> $i")
    method()
}
```
转成Java
```java
   public static final void main() {
      int i$iv = 100;
      int $i$f$test = false;
      String var2 = "transport -> " + i$iv;
      boolean var3 = false;
      System.out.println(var2);
      int var4 = false;
      String var5 = "haha";
      boolean var6 = false;
      System.out.println(var5);                   //<---- 没有输出xyz
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void test(int i, @NotNull Function0 method) {
      int $i$f$test = 0;
      Intrinsics.checkParameterIsNotNull(method, "method");
      String var3 = "transport -> " + i;
      boolean var4 = false;
      System.out.println(var3);
      method.invoke();
   }
```
加上`crossinline`后
```kotlin
fun main() {
    test(100) {
        println("haha")
        return@test                                   //<---- 这边使用return会报错，只能返回当前lambda
    }
    println("xyz")
}

inline fun test(i: Int, crossinline method: () -> Unit){
    println("transport -> $i")
    method()
}
```
转成Java
```java
   public static final void main() {
      int i$iv = 100;
      int $i$f$test = false;
      String var2 = "transport -> " + i$iv;
      boolean var3 = false;
      System.out.println(var2);
      int var4 = false;
      String var5 = "haha";
      boolean var6 = false;
      System.out.println(var5);
      String var7 = "xyz";
      $i$f$test = false;
      System.out.println(var7);                         //<---- 输出xyz
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void test(int i, @NotNull Function0 method) {
      int $i$f$test = 0;
      Intrinsics.checkParameterIsNotNull(method, "method");
      String var3 = "transport -> " + i;
      boolean var4 = false;
      System.out.println(var3);
      method.invoke();
   }
```
该关键字需要在`inline`修饰的方法中使用
