# 概念
- **主线程**

也叫`UI线程、MainThread`，当程序第一次启动时自动开启一条主线程，用于处理UI操作（界面更新等）
- **子线程**

也叫`工作线程`，手动开启，比如`new Thread().start()`，用于处理耗时操作（网络请求等）
- **消息处理者（Handler）**

主线程与子线程的通信媒介，线程`Message`的主要处理者，用于添加`Message`到`MessageQueue`，也用于处理`Looper`分发的`Message`
- **消息（Message）**

用于线程间通信，比如子线程向主线程发送消息更新页面
- **消息队列（MessageQueue）**

数据结构，用于存储`Handler`发过来的`Message`
- **轮循器（Looper）**

`MessageQueue`和`Handler`的通信媒介，用于`Message`的获取(循环取出`MessageQueue`的`Message`)和分发(将取出的`Message`发送给对应的`Handler`)


**注意：**
- 一个线程只能绑定一个Looper，但可以有多个Handler
- 一个Looper能绑定多个线程的Handler，即**多个线程能发消息给同一个Looper持有的`MessageQueue`**
- 一个Handler只能绑定一个Looper

# 描述
在Android开发中，为了保证UI操作是线程安全的，规定只允许UI线程更新UI组件，但在实际开发中可能会出现多条线程并发操作更新UI的情况，
这就导致UI操作的线程不安全，与规定冲突。为了解决该冲突，Android提供了Handler消息传递机制，当工作线程需要更新UI时通过Handler通知UI线程，从而在UI线程更新UI

# 相关类
在该机制中有3个重要类
- Handler
  - public final boolean sendMessage(Message msg)
  - public void handleMessage(Message msg)
  - public void dispatchMessage(Message msg)
  - public final boolean post(Runnable r)
- MessageQueue
  - boolean enqueueMessage(Message msg, long when)//插入Message
  - Message next()//读取Message，该操作同时会伴随删除操作，该方法是无限循环的方法，如果队列中没有消息则会一直阻塞
- Looper
  - public static void prepare()
  - public static void loop()
  
# 使用
Handler的使用在Android开发中非常常见，因发送消息到消息队列的方式不同而不同
- sendMessage()
- post()
## sendMessage()
### 自定义Handler类继承自Handler，重写`handMessage`方法
```java
public class MyHandler extends Handler{
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
    }
}
```
### 匿名内部类
```java
private Handler handler = new Handler(){
    @Override
    public void handleMessage(@NonNull Message msg) {
        super.handleMessage(msg);
    }
};
```
发送消息可以封装一个方法
```java
private void sendMsg(Handler handler, int what, Object obj){
    Message message = handler.obtainMessage();
    message.what = what;
    message.obj = obj;
    handler.sendMessage(message);
}
```
## post()
```java
handler.post(new Runnable() {
    @Override
    public void run() {
        //要执行的UI操作
    }
});
```
# 工作流程
0. 程序启动
1. 主线程创建`Looper、MessageQueue、Handler`。这三者皆属于主线程，创建消息队列后，Looper自动进入消息循环，Handler绑定了主线程的Looper和消息队列
2. 工作线程通过`Handler`发送`Message`到`MessageQueue`。消息内容即为该工作线程想对UI进行的操作
3. Looper循环，将`Message`取出（消息出队），然后Looper将取出的`Message`发送给创建该消息的`Handler`
4. Handler接收Looper发送过来的`Message`，根据该Message进行UI操作

Handler创建时候会采用当前线程的Looper来构造消息循环系统，需要注意的是，线程默认是没有Looper的，如果在一个子线程中与另一个子线程通信直接使用Looper会报错，这时候
就需要为线程创建Handler，默认的UI线程(ActivityThread)在被创建的时候就会初始化Looper，这也是在主线程中默认可以使用Handler的原因
