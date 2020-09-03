众所周知，Activity的启动模式有四种：
- standard
- singleTop
- singleTask
- singleInstance

为什么要有启动模式？对于这个问题，可以看看微信，我们从聊天页面点击头像，进入好友信息页面，可以看到下面还有个发消息的按钮。如果没有修改启动模式，那么每次页面跳转都会创建一个聊天的Activity，而之前的一直都在，没有销毁（除非手动finish）。当然，这只是个人的猜测。下面就具体说一下Activity启动模式。

在此之前，需要了解一件事：Android采用Task管理多个Activity。当我们启动一个应用（这里指的是冷启动）时，Android就会为这个应用创建一个Task，事实上可以把这个Task理解为管理Activity的栈：先启动的Activity在栈底，后启动的在栈顶。

# standard
当未指定启动模式时，默认使用该模式来启动Activity。Android总会为目标Activity创建一个新的实例，并将该实例Activity添加到当前Task栈中。举个例子：在MainActivity中启动MainActivity，每次都会进入一个新的MainActivity页面。当点击back键时，系统会挨个删除栈顶Activity

# singleTop
与standard基本一致，有一点不同：若要启动的Activity实例已经在Task栈顶，则不会创建新的实例，而是直接复用；若要启动的Activity实例不在栈顶，会重新创建实例，并加入到栈顶。举个例子：修改上述例子的启动模式，再次在MainActivity中启动MainActivity，会发现无变化

# singleTask
该模式确保在同一个Task内只有一个实例，分以下三种情况：
- 如果目标Activity不存在，则创建实例，加入到栈顶
- 如果目标Activity存在且位于栈顶，则不创建新实例，直接复用
- 如果目标Activity存在但不在栈顶，则把该实例之上的所有Activity移出栈，进而使自己成为栈顶

# singleInstance
系统保证无论从哪个Task启动目标Activity，只会创建一个目标Activity实例，并使用一个全新的Task加载该实例。分以下两种情况：
- 如果目标Activity不存在，则创建一个新的Task，再创建目标Activity实例，然后加入到新的Task栈顶
- 如果目标Activity已存在，则无论它位于哪个应用程序、哪个Task，系统都会把该Activity所在的Task转到前台，从而显示Activity
