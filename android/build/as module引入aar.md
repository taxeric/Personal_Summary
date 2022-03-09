## 北极狐,熊峰以下版本
1. module层的libs加入aar
2. project层的build.gradle的`allprojects - repositories`编写如下代码
```kotlin
allprojects {
    repositories {
        ...
        flatDir {
            dirs project(":引入aar的module名字").file("一般是libs,除非你放aar到别的文件夹了")
        }
    }
}
```
3. 如果引入module的其他模块想使用该aar则: 
```kotlin
api (name: "aar名字", ext: "aar")
```
4. 如果引入module的其他模块不使用该aar则将上述`api`换成`implementation`即可
