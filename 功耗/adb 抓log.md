## 以下`findstr`可替换为`grep`，若后者提示`xxx不是内部或外部命令`可以用前者
找指定进程pid
```
$ adb shell ps | findstr 包名
这里的findstr是因为grep未找到
```
输出指定级别log
```
$ adb logcat *:E
E可替换成V、I等
```
抓指定TAG
```
$ adb logcat | findstr 程序TAG
$ adb logcat -v 程序TAG
两种方式试试，写这篇文档时候没试
```
抓指定进程的指定级别log
```
举例：抓E
找到PID后输入一下命令
$ adb logcat *:E process | findstr 进程PID
```
