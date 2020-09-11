| 测试机型 | OPPO A3 |
|--|--|
| Android版本 | 9.0 |
| AS版本 | 4.2.0 |
## 检查权限

首先要判断通知权限是否开启
参考　[Android 检查通知权限](https://www.jianshu.com/p/1e27efb1dcac)

## 设置渠道

对于安卓版本 < 8.0的情况网上已经有成熟的案例了，故主要看安卓版本 >= 8.0的情况，该情况下**发送通知的时候必须要为通知设置通知渠道，否则通知不会被发送**

> 创建通知渠道后，您便无法更改通知行为，此时用户拥有完全控制权。不过，您仍然可以更改渠道的名称和说明

设置渠道的代码为
```java
notificationManager.createNotificationChannel(
        new NotificationChannel(
                channelID, 
                channelName, 
                NotificationManager.IMPORTANCE_HIGH
        )
);
```
| 参数 | 意义 |
|--|--|
| channelID | 渠道ID，要保证该ID全局唯一 |
| channelName | 渠道名称，用户可以直接看到 |
| importance | 重要等级 |

关于`importance`有如下几个等级 (在NotificationManager中定义)，对应的值分别从　０　－　５

- IMPORTANCE_NONE
- IMPORTANCE_MIN
不发出提示音，且不会在状态栏中显示
- IMPORTANCE_LOW
不发出提示音
- IMPORTANCE_DEFAULT
发出提示音
- IMPORTANCE_HIGH
发出提示音，并以浮动通知的形式显示
- IMPORTANCE_MAX

> 无论重要性级别如何，所有通知都会在非干扰系统界面位置显示

**自定义渠道**

channelID = "自定义id"
channelName = "自定义name"

Notification.Builder().setChannelId("自定义id")

举个例子

```java
    private void create(){
        NotificationManager manager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
        Notification.Builder builder = new Notification.Builder(MainActivity.this);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O){
            String channelID = "minecraft";
            String channelName = "我的世界";
            if (manager != null){
                manager.createNotificationChannel(new NotificationChannel(channelID, channelName, NotificationManager.IMPORTANCE_HIGH));
            }
            builder.setChannelId(channelID);
        }

        builder.setContentTitle("标题")
                .setContentText("文本")
                .setSmallIcon(R.mipmap.ic_launcher)
                .build();
        manager.notify(1, builder.build());
    }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020061611361639.png)可以看到在应用的通知里面已经有自定义的渠道名称了


## 常见方法
- setStyle
该方法被用于创建展开式通知

| 子类 | 说明 | 方法 |
|--|--|--|
| BigPictureStyle | 该实例用于添加大图片，要使该图片仅在通知收起时显示为缩略图，需调用 setLargeIcon() 并传入该图片，同时调用 BigPictureStyle.bigLargeIcon() 并传入 null，这样大图标就会在通知展开时消失 | 见　https://developer.android.google.cn/training/notify-user/expanded |
| BigTextStyle | 该实例用于添加大段文本，以在通知的展开内容区域显示文本 | 见　https://developer.android.google.cn/training/notify-user/expanded |
| InBoxStyle | 该实例用于添加简短的摘要行，即每条文本被截断成一行，而不是显示为像BigTextStyle一样的连续文本。要添加新行，最多可调用 addLine() 6 次。如果添加的行超过 6 行，则仅显示前 6 行 | 见　https://developer.android.google.cn/training/notify-user/expanded |
| MessageStyle | 该实例用于显示对话，通过单独处理发送人姓名和消息文本为每条消息提供一致的布局 | 见　https://developer.android.google.cn/training/notify-user/expanded |
| MediaStyle | 该实例用于显示媒体播放控件和曲目信息，最多可调用 addAction() 5 次，以显示最多 5 个单独的图标按钮 | 见　https://developer.android.google.cn/training/notify-user/expanded |

- setAutoCancel
该方法在用户点按通知后自动移除通知


## 设置点击事件
使用`PendIntent`，`PendIntent`和`Intent`的区别：后者是立即发生，前者是在未来某个时刻发生

>PendIntent 支持三种意图
> - PendingIntent.getActivity(Context context,int  requestCode,Intent intent,int flags)，该待定意图发生时，效果相当于Context.startActivity(Intent)
>- PendingIntent.getService(Context context,int requestCode,Intent intent,int flags)，该待定意图发生时，效果相当于Context.startService(Intent)
>- PendingIntent.getBroadcast(Context context,int requestCode,Intent intent,int flags)，该待定意图发生时，效果相当于Context.sendBroadcast(Intent)

**flags参数**

该参数有４种类型

- FLAG_ONE_SHOT
在所有相同的PendingIntent中只有第一个被使用的生效，其余不生效
- FLAG_NO_CREATE
如果要创建的PendingIntent尚未存在，则不创建新的PendingIntent，直接返回null
- FLAG_CANCEL_CURRENT
在所有相同的PendingIntent中只有最新的那个可以使用
- FLAG_UPDATE_CURRENT
所有相同的PendingIntent 都会更新Extra的内容与最新的保持一致，并且都可以使用

此外可以传入０，表示不用任何 flags 控制 PendIntent 的创建

> PendingIntent重写了equals方法，如果我们说两个PendingIntent是相同的，那么也就是说，它们封装的Intent是相同的并且requestCode也是相同的。注意，这里我们说的Intent相同并不是说是相同的Intent对象，而是指两个intent具有相同的action、data、categories、components、type和flags（这个flags是intent的flags）。因为在比较PendingIntent中封装的intent时，调用的是Intent的filterEquals方法，而该方法认为只要两个intent具有相同的action、data、categories、components、type、flags就认为它们两个是相同的

## 显示通知
```java
notificationManager.notify(notificationId, builder.build());
```
