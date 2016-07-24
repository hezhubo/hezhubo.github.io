---
title: ADB常用命令
date: 2016-07-06 23:35:56
tags: [android,adb]
---
#### 常用命令
##### adb帮助
```bash
adb help
```
</br>
##### 启动adb服务
```bash
adb start-server
```
</br>
##### 终止adb服务
```bash
adb kill-server
```
</br>
##### 显示当前运行的全部设备
```bash
adb devices [-l]　　// [-l] 显示设备详细信息
```
</br>
##### 对某一设备执行命令
```bash
adb -s <specific device>　　// <specific device> 设备ID
```
</br>
##### 重启设备
```bash
adb reboot
```
</br>
##### 以网络方式连接设备
```bash
adb connect <host>[:<port>]　　// 设备ip+端口 默认5555
```
</br>
##### 断开网络设备
```bash
adb disconnect [<host>[:<port>]]　　// 设备ip+端口 默认5555
```
</br>
##### 安装（覆盖）应用
```bash
adb install -r <file>　　// <file> apk路径
```
</br>
##### 卸载应用
```bash
adb uninstall <package>　　// <package> 应用包名
```
</br>
##### 传输文件到设备
```bash
adb push <local>... <remote>　　// <local> 本地文件路径  <remote> 设备保存路径
```
</br>
##### 拉取设备文件到电脑
```bash
adb pull <remote>... <local>　　// <local> 电脑保存路径  <remote> 设备文件路径
```
</br>
##### 拉取设备文件到电脑
```bash
adb pull <remote>... <local>　　// <local> 电脑保存路径  <remote> 设备文件路径
```
</br>
##### 打印日志
```bash
adb logcat --help　　// logcat帮助
adb logcat [options] [filterspecs]　　// [options] [filterspecs] 过滤参数
```
</br>
###### 清楚缓冲区日志
```bash
adb logcat -c
```
</br>
###### 输出日志到文件
```bash
adb logcat -f <filename>　　// <filename>文件路径(手机上的路径)
adb logcat -d -f <filename>　　// 边输出到屏幕，边保存
```
</br>
###### 指定日志输出格式
```bash
adb logcat -v time　　// 带时间的输出
adb logcat -v threadtime　　// 带时间和线程信息
```
</br>
###### 过滤标签
```bash
adb logcat -s hezb　　// 过滤TAG为 hezb 的所以log
adb logcat -s hezb System.out　　// 过滤多个TAG(hezb, System.out)
```
</br>
###### 按等级过滤
*优先级从低到高：*
Log.v  - VERBOSE  : 黑色
Log.d  - DEBUG     : 蓝色
Log.i   - INFO         : 绿色
Log.w - WARN      : 橙色
Log.e  - ERROR     : 红色
```bash
adb logcat *:D　　// 过滤Debug以上优先级的log
adb logcat hezb:D *:S　　// 过滤TAG为 hezb Debug以上优先级的log,没有 *:S 则无法正确输出
adb logcat hezb:D test:I *:S　　// 过滤多个TAG(hezb,test),并且各自优先级可选
```
</br>
###### 使用系统命令过滤
```bash
adb logcat | grep "hezb"　　// linux 使用 grep 命令，详细过滤看grep用法
adb logcat | find "hezb"　　// windows 使用 find 命令
```
</br>

##### 登陆设备
```bash
adb shell
```
执行完shell后，可使用android支持的linux命令
</br>