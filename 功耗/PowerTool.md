# 使用
1. 开电源，正常黄灯，非正常橙黄灯
2. Set Vout（可选），设置电压，一般custom
3. Vout Enable```开```输出电压
4. 贴电池，有黄色胶布那个，贴到电池框上面，像sim卡的那个位置旁边
5. 镊子同时按着GND和PWR开机（若没有开机，一般没开机）
6. PowerTool点RUN开始测电流
7. 熄屏转到步骤5
8. 测试一般5min
9. 测完先Vout Enable```关```输出电压
10. 关电源

# 刷机
**重要**  
**QFIL用来刷工厂版本，没有MIUI框架，MiFlash用来刷小米系统**  
**需要提前准备好文件**  

**MIUI版本目录：共享文件夹/path/miui/xxxxx**  
**工厂版本目录：共享文件夹/path/daily/xxxxx**

0. 若没拔黄色胶布连的电源先拔了
1. 镊子同时按着GND和BT进入紧急xx模式（或者输入```adb reboot edl```命令进入9008模式）
2. 在一分钟或更短时间内刷机
3. 若用的QFIL则显示Finish Download表示完成，若用的MiFlash看Log，显示```after restart close COM3```表示完成，后者一般需要几分钟，等着就行
