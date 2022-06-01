---
title: ADB常用命令
date: 2016-07-06 23:35:56
tags: [android,adb]
categories: Android
---
#### 常用命令
##### adb帮助
```bash
adb help
```
##### 启动adb服务
```bash
adb start-server
```
##### 终止adb服务
```bash
adb kill-server
```
##### 显示当前运行的全部设备
```bash
adb devices [-l]　　// [-l] 显示设备详细信息
```
##### 对某一设备执行命令
```bash
adb -s <specific device>　　// <specific device> 设备ID
```
##### 重启设备
```bash
adb reboot
```
##### 以网络方式连接设备
```bash
adb connect <host>[:<port>]　　// 设备ip+端口 默认5555
```
##### 断开网络设备
```bash
adb disconnect [<host>[:<port>]]　　// 设备ip+端口 默认5555
```
##### 安装（覆盖）应用
```bash
adb install -r <file>　　// <file> apk路径
```
##### 卸载应用
```bash
adb uninstall <package>　　// <package> 应用包名
```
##### 传输文件到设备
```bash
adb push <local>... <remote>　　// <local> 本地文件路径  <remote> 设备保存路径
```
##### 拉取设备文件到电脑
```bash
adb pull <remote>... <local>　　// <local> 电脑保存路径  <remote> 设备文件路径
```
##### 打印日志
```bash
adb logcat --help　　// logcat帮助
adb logcat [options] [filterspecs]　　// [options] [filterspecs] 过滤参数
```
###### 清除缓冲区日志
```bash
adb logcat -c
```
###### 输出日志到文件
```bash
adb logcat -f <filename>　　// <filename>文件路径(手机上的路径)
adb logcat -d -f <filename>　　// 边输出到屏幕，边保存
```
###### 指定日志输出格式
```bash
adb logcat -v time　　// 带时间的输出
adb logcat -v threadtime　　// 带时间和线程信息
```
###### 过滤标签
```bash
adb logcat -s hezb　　// 过滤TAG为 hezb 的所以log
adb logcat -s hezb System.out　　// 过滤多个TAG(hezb, System.out)
```
###### 按等级过滤
> *优先级从低到高：*
> Log.v  - VERBOSE  : 黑色
> Log.d  - DEBUG     : 蓝色
> Log.i   - INFO         : 绿色
> Log.w - WARN      : 橙色
> Log.e  - ERROR     : 红色

```bash
adb logcat *:D　　// 过滤Debug以上优先级的log
adb logcat hezb:D *:S　　// 过滤TAG为 hezb Debug以上优先级的log,没有 *:S 则无法正确输出
adb logcat hezb:D test:I *:S　　// 过滤多个TAG(hezb,test),并且各自优先级可选
```
###### 使用系统命令过滤
```bash
adb logcat | grep "hezb"　　// linux 使用 grep 命令，详细过滤看grep用法
adb logcat | find "hezb"　　// windows 使用 find 命令
```
##### 登陆设备
```bash
adb shell　　// 普通用户权限
adb root　　// root用户权限(需要手机已经root)
```
###### shell权限下
```bash
su　　// 切换成root用户(需要手机已经root)
```
###### root权限下
```bash
mount -o remount / 　　// 挂载 系统文件夹 到当前目录
rm xxx.apk　　// 删除系统应用
```
###### 模拟键盘鼠标事件
```bash
input keyevent <value>　　// 模拟键盘按键 value对应键盘键值
sendevent [device] [type] [code] [value]　　// 全事件，需要多个组合使用  device: /dev/input/event0 
getevent　　// 监听当前手机事件
```
键盘事件表

| KeyEvent Value | KEYCODE               |
| :------------- | :-------------------- |
| 0              | KEYCODE_UNKNOWN       |
| 1              | KEYCODE_MENU          |
| 2              | KEYCODE_SOFT_RIGHT    |
| 3              | KEYCODE_HOME          |
| 4              | KEYCODE_BACK          |
| 5              | KEYCODE_CALL          |
| 6              | KEYCODE_ENDCALL       |
| 7              | KEYCODE_0             |
| 8              | KEYCODE_1             |
| 9              | KEYCODE_2             |
| 10             | KEYCODE_3             |
| 11             | KEYCODE_4             |
| 12             | KEYCODE_5             |
| 13             | KEYCODE_6             |
| 14             | KEYCODE_7             |
| 15             | KEYCODE_8             |
| 16             | KEYCODE_9             |
| 17             | KEYCODE_STAR          |
| 18             | KEYCODE_POUND         |
| 19             | KEYCODE_DPAD_UP       |
| 20             | KEYCODE_DPAD_DOWN     |
| 21             | KEYCODE_DPAD_LEFT     |
| 22             | KEYCODE_DPAD_RIGHT    |
| 23             | KEYCODE_DPAD_CENTER   |
| 24             | KEYCODE_VOLUME_UP     |
| 25             | KEYCODE_VOLUME_DOWN   |
| 26             | KEYCODE_POWER         |
| 27             | KEYCODE_CAMERA        |
| 28             | KEYCODE_CLEAR         |
| 29             | KEYCODE_A             |
| 30             | KEYCODE_B             |
| 31             | KEYCODE_C             |
| 32             | KEYCODE_D             |
| 33             | KEYCODE_E             |
| 34             | KEYCODE_F             |
| 35             | KEYCODE_G             |
| 36             | KEYCODE_H             |
| 37             | KEYCODE_I             |
| 38             | KEYCODE_J             |
| 39             | KEYCODE_K             |
| 40             | KEYCODE_L             |
| 41             | KEYCODE_M             |
| 42             | KEYCODE_N             |
| 43             | KEYCODE_O             |
| 44             | KEYCODE_P             |
| 45             | KEYCODE_Q             |
| 46             | KEYCODE_R             |
| 47             | KEYCODE_S             |
| 48             | KEYCODE_T             |
| 49             | KEYCODE_U             |
| 50             | KEYCODE_V             |
| 51             | KEYCODE_W             |
| 52             | KEYCODE_X             |
| 53             | KEYCODE_Y             |
| 54             | KEYCODE_Z             |
| 55             | KEYCODE_COMMA         |
| 56             | KEYCODE_PERIOD        |
| 57             | KEYCODE_ALT_LEFT      |
| 58             | KEYCODE_ALT_RIGHT     |
| 59             | KEYCODE_SHIFT_LEFT    |
| 60             | KEYCODE_SHIFT_RIGHT   |
| 61             | KEYCODE_TAB           |
| 62             | KEYCODE_SPACE         |
| 63             | KEYCODE_SYM           |
| 64             | KEYCODE_EXPLORER      |
| 65             | KEYCODE_ENVELOPE      |
| 66             | KEYCODE_ENTER         |
| 67             | KEYCODE_DEL           |
| 68             | KEYCODE_GRAVE         |
| 69             | KEYCODE_MINUS         |
| 70             | KEYCODE_EQUALS        |
| 71             | KEYCODE_LEFT_BRACKET  |
| 72             | KEYCODE_RIGHT_BRACKET |
| 73             | KEYCODE_BACKSLASH     |
| 74             | KEYCODE_SEMICOLON     |
| 75             | KEYCODE_APOSTROPHE    |
| 76             | KEYCODE_SLASH         |
| 77             | KEYCODE_AT            |
| 78             | KEYCODE_NUM           |
| 79             | KEYCODE_HEADSETHOOK   |
| 80             | KEYCODE_FOCUS         |
| 81             | KEYCODE_PLUS          |
| 82             | KEYCODE_MENU          |
| 83             | KEYCODE_NOTIFICATION  |
| 84             | KEYCODE_SEARCH        |
| 85             | TAG_LAST_KEYCODE      |
