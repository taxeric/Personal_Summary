是Android的锁机制，表现为只要进程持有该锁，系统就无法进入休眠
### 申请
在AndroidManifest.xml写
```xml
android.Manifest.permission.WAKE_LOCK 
```
### 分类
#### 作用时间分类：
- 永久锁  
只要获取了该锁，就必须显式释放
- 超时锁  
在达到预定时间后自动释放
#### 释放原则分类
- 计数锁  
一次申请对应一次释放
- 非计数锁  
n次申请对应一次释放

**如果申请WakeLock，则必须有PowerManager实例**
```java
 PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
 PowerManager.WakeLock wl = pm.newWakeLock(PowerManager.SCREEN_DIM_WAKE_LOCK, "My Tag");
 wl.acquire();
 Wl.acquire(int timeout);//超时锁
```
释放
```java
wl.release();
```

