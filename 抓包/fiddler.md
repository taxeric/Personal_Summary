## 抓app包
1. 笔记本开热点
2. 打开Fidder，点击菜单栏中的 [Tools] –> [Fiddler Options]
3. 点击 [Connections] ，设置代理端口是8888， 勾选 Allow remote computers to connect， 点击OK
4. 这时在 Fiddler 可以看到自己本机无线网卡的IP了（要是没有的话，重启Fiddler，或者可以在cmd中ipconfig找到自己的网卡IP）
5. 在手机端连接PC的wifi，并手动设置代理IP与端口（代理IP就是上图的IP，端口是Fiddler的代理端口8888）
6. 访问网页输入代理IP和端口，下载Fiddler的证书，FiddlerRoot certificate
7. 安装完了证书，可以用手机访问应用，就可以看到截取到的数据包了。



https://www.cnblogs.com/yyhh/p/5140852.html
