---
title: 解决ADB 5037端口 被外部程序占用问题
date: 2016-07-05 00:44:52
tags: [android,adb]
---
**windows环境下：**
``` python
netstat -ano | findstr "5037"
```
结果：
 TCP    127.0.0.1:5037         0.0.0.0:0              LISTENING       **1408**
 ...

查看哪个进程占用了
``` python
tasklist | findstr "1408"
```
结果：
SogouPhoneService.exe         1408 Console                    1      8,836 K

原来是SogouPhoneService进程占了adb的端口

用命令或打开任务管理器终止掉该进程就可以了。

再执行
``` python
adb devices
```
OK!!!