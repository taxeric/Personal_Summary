建议放在app私有目录
```java
/storage/emulated/0/Android/data/com.xxx.xxx/xxxx
```
||||
|--|--|--|
|私有目录名字|方法s|方法|
|Media|getExternalMediaDirs()|无|
|Obb|getObbDirs()|getObbDir()|
|Cache|getExternalCacheDirs()|getExternalCacheDir()|
|Data|getExternalFilesDirs(type)|getExternalFilesDir(type)|

其中Data目录`type`可为null，主要有如下值
- android.os.Environment.DIRECTORY_MUSIC
- android.os.Environment.DIRECTORY_PODCASTS
- android.os.Environment.DIRECTORY_RINGTONES
- android.os.Environment.DIRECTORY_ALARMS
- android.os.Environment.DIRECTORY_NOTIFICATIONS
- android.os.Environment.DIRECTORY_PICTURES
- android.os.Environment.DIRECTORY_MOVIES

#### 举例
包名：com.eric.wanandroid
```kotlin
val path = externalCacheDir
LogUtils.i("filePath ${path?.toString()}")

输出
filePath /storage/emulated/0/Android/data/com.eric.wanandroid/cache
```

除了`getObbDir()`方法外，其余方法如果文件夹不存在则自动创建；obb目录在/storage/emulated/0/Android/obb
