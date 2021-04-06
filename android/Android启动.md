1. 电源启动，加载BootLoader到RAM（运存）
2. BootLoader拉起系统OS
3. 启动Linux内核，会找init.rc文件
4. 启动init进程
5. 启动Zygote进程，创建Java虚拟机并注册jni方法，创建服务端socket
6. 启动SystemServer进程，启动Binder线程池和SystemServiceManager，启动各种系统服务
7. 启动Launcher，显示已安装的应用图标

