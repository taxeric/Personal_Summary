文档：[Battery Historian使用](https://developer.android.google.cn/topic/performance/power/battery-historian?hl=zh_cn)

**以下操作在开发者模式下运行，需要连接usb**
1. kill server（可选）
```java
adb kill-server
```
2. start server（可选）
```java
adb start-server
```
3. 看设备是否存在（可选）
```java
adb devices
```
4. 重置电量统计
```java
adb shell dumpsys batterystats --reset
```
5. 拔下usb，保持只用电池耗电
6. 做操作，比如浏览皮皮虾
7. 再插上usb
8. 导出统计
```java
adb bugreport > [目录]bugreport.zip
目录指的是存储目录，比如D盘，若没写，就是cmd打开的路径
```

查看当前显示的应用包名及触摸焦点
```java
adb shell dumpsys window | findstr mCurrentFocus
```

