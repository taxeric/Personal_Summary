#### A创建
```java
    onCreate
    onStart
    onResume
```
#### A -> B
```java
A:  onPause
B:  onCreate
    onStart
    onResume
A:  onStop
```
#### B返回A
```java
B:  onPause
A:  onRestart
    onStart
    onResume
B:  onStop
    onDestory
```
#### A -> 透明的 B
```java
A:  onPause
B:  onCreate
    onStart
    onResume
```
#### 透明的B返回A
```java
B:  onPause
A:  onResume
B:  onStop
    onDestory
```
#### 创建普通Dialog
无变化
#### 创建系统级别Dialog
触发onPause

