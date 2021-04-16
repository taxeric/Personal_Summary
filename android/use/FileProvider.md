# 前言
从 Android N（7.0） 开始，将严格执行 StrictMode 模式，也就是说，将对安全做更严格的校验。而从 Android N 开始，将不允许在 App 间，使用 file:// 的方式，传递一个 File ，否者会抛出 FileUriExposedException 的错误，会直接引发 Crash。

但是，既然官方对文件的分享做了一个这么强硬的修改（直接抛出异常），实际上也提供了解决方案，那就是 FileProvider，通过 content:// 的模式替换掉 file:// ，同时，需要开发者主动升级 targetSdkVersion 到 24 才会执行此策略。

# 常用场景
之前做应用更新时候接触到了这个东西

FileProvider使用场景大致如下
- 相机拍照
- 应用安装

# 如何使用
FileProvider存在与V4包中，是ContentProvider的子类，故需要在Androidmanifest文件中注册

```xml
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.demo.testdemo.provider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
```
## 参数说明
| 参数 | 说明 |
|--|--|
| name | V4包的FileProvider，也可为其实现类 |
| authorities | Uri域名，可以随便写，一般以.fileprovider结尾；**为当前系统唯一值，不能与其他APP一致** |
| exported | 该 FileProvider 是否需要公开 |
| grantUriPermissions | 是否允许授权文件的临时访问权限 |

## 注意事项
写法基本是固定的，尽管很多项都可以修改，但不建议这么做

**如果是Androidx版本，name需要更改为androidx.core.content.FileProvider**

对于`<meta-data/>`标签，用于配置FileProvider支持分享出去的目录。其中name是固定的，resource指向xml文件

## paths标签
如果没有res-xml目录，需要手动创建。创建完成后新建file_paths.xml文件，当然名字可以随便写，这里只是与上面一致。

file_paths.xml内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    ......
</paths>
```
在`paths`标签内至少需要配置一个`xxx-path`标签。例如

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path
        name="install_app"
        path="."/>
</paths>
```
该标签表示：当前FileProvider使用Environment.getExternalStorageDirectory()指向的目录

path：临时授权的访问路径
name：路径的名字，唯一值，不重复

此外还有其他标签，在FileProvider类中可以查阅。如下：

```java
    private static final String TAG_ROOT_PATH = "root-path";
    private static final String TAG_FILES_PATH = "files-path";
    private static final String TAG_CACHE_PATH = "cache-path";
    private static final String TAG_EXTERNAL = "external-path";
    private static final String TAG_EXTERNAL_FILES = "external-files-path";
    private static final String TAG_EXTERNAL_CACHE = "external-cache-path";
```
对应指向的目录如下：

```java
File target = null;
if (TAG_ROOT_PATH.equals(tag)) {
    target = DEVICE_ROOT;
} else if (TAG_FILES_PATH.equals(tag)) {
    target = context.getFilesDir();
} else if (TAG_CACHE_PATH.equals(tag)) {
    target = context.getCacheDir();
} else if (TAG_EXTERNAL.equals(tag)) {
    target = Environment.getExternalStorageDirectory();
} else if (TAG_EXTERNAL_FILES.equals(tag)) {
    File[] externalFilesDirs = ContextCompat.getExternalFilesDirs(context, null);
	if (externalFilesDirs.length > 0) {
        target = externalFilesDirs[0];
	}
} else if (TAG_EXTERNAL_CACHE.equals(tag)) {
    File[] externalCacheDirs = ContextCompat.getExternalCacheDirs(context);
    if (externalCacheDirs.length > 0) {
        target = externalCacheDirs[0];
    }
}
```
- root-path：表示根目录
- files-path：表示 context.getFileDir() 获取到的目录
- cache-path：表示 context.getCacheDir() 获取到的目录
- external-path：表示Environment.getExternalStorageDirectory() 指向的目录
- external-files-path：表示 ContextCompat.getExternalFilesDirs() 获取到的目录
- external-cache-path：表示 ContextCompat.getExternalCacheDirs() 获取到的目录

## 使用content://
FileProvider类提供getUriForFile方法创建Uri，调用该方法后，会得到一个`file://`转换成`content://`的uri对象，直接使用该对象即可

```java
    public static Uri getUriForFile(Context context, String authority, File file) {
        final PathStrategy strategy = getPathStrategy(context, authority);
        return strategy.getUriForFile(file);
    }
```
其中`authority`就是在Androidmanifest.xml配置的authority，即两者值一致

# 其他事项
## 节省代码
在项目开发中不必为了FileProvider而导入整个V4包，可以直接复制FileProvider的代码，简单修改能运行就行
## 友好使用
 如果是开发Lib等，需要动态修改`authority`的值，友好使用该Lib

### 例子
你开发了一个Lib，包名为`com.install.demo`，内部实现了安装APP的功能
```java
public static void installApp(File file, Context context){
	Intent intent = new Intent(Intent.ACTION_VIEW);
	intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
	Uri uri = FileProvider.getUriForFile(context, BuildConfig.APPLICATION_ID + ".fileprovider", file);
	intent.setDataAndType(uri, "application/vnd.android.package-archive");
	context.startActivity(intent);
}
```
而使用你Lib的用户包名为`com.demo.test`，故最终Lib和用户的APPLICATION_ID不一致，会导致程序抛出异常

将该方法修改如下即可：
```java
public static void installApp(File file, Context context){
	...
	Uri uri = FileProvider.getUriForFile(context, context.getPackageName() + ".fileprovider", file);
	...
}
```
## 多FileProvider情况
如果你导入了第三方的库，在该库的清单文件中已经注册了FileProvider，如下：

```xml
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.install.demo.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/xxx" />
        </provider>
```

而你的项目中又要使用到FileProvider，那么你就要自己写一个类，继承FileProvider，将name替换为继承自FileProvider的子类。如下：
```java
public class MyProvider extends FileProvider {
}
```
无需实现代码，就这样即可。接着修改你的清单，如下：
```xml
        <provider
            android:name="com.demo.test.MyProvider"
            .../>
        </provider>
```
也适用于在一个项目中使用多个FileProvider的情况
