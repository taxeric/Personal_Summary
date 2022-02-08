在 Android 12 中包含`<intent-filter>`的`activity`、`service`或`receiver`必须为这些应用组件显示声明`android:exported`属性
```xml
<activity
    android:name=".TestActivity"
    android:exported="false">
    <intent-filter>
        ......
    </intent-filter>
</activity>
```
如果在包含`<intent-filter>`的`activity`、`service`或`receiver`组件中，没有显示声明`android:exported`的值，你的应用将无法安装
```
Installation did not succeed.
The application could not be installed: INSTALL_FAILED_VERIFICATION_FAILURE
List of apks:
[0] '.../build/outputs/apk/debug/app-debug.apk'
Installation failed due to: 'null'
```
如果您的应用在需要声明`android:exported`的值时未进行此声明
```
Targeting S+ (version 10000 and above) requires that an explicit value for \
android:exported be defined when intent filters are present
```

## 为什么要声明
android:exported 属性的默认值取决于是否包含 <intent-filter>，如果包含 <intent-filter> 那么默认值为 true，否则 false
- 当 android:exported="true" 时，如果不做任何处理，可以接受来自其他 App 的访问
- 当 android:exported="false" 时，限制为只接受来自同一个 App 或一个具有相同 user ID 的 App 的访问
