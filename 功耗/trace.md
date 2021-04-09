需要```android sdk```，具体为```systrace-systrace.py```文件
## 记录trace文件
*must!重要*：需要python环境，2.x  
命令行
```java
python systrace.py -o mmTrace.html -t 10
```
参数：  
- o：名字
- t：记录时长，单位秒
- 更多命令查看[文档](https://developer.android.google.cn/topic/performance/tracing/command-line)  
会默认保存在systrace目录下
## 查看trace文件
1. 谷歌浏览器，输入chrome://tracing/  导入文件
2. https://ui.perfetto.dev/#!/viewer  导入文件，相当于上述2.0版本


使用1.0版的trace，上面可能有带"F"字样的圆，在```16.6```毫秒内渲染的必须保持每秒```60```帧稳定帧速率的帧会以绿色圆圈表示。渲染时间超过```16.6```毫秒的帧会以黄色或红色帧圆圈表示。

