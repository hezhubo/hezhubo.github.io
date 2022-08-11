---
title: Flutter的一些尝试
date: 2022-08-09 18:52:45
categories: Flutter
---

# Flutter的一些尝试

### 一、环境部署

> 官方链接：https://flutter.cn/docs/get-started/install

官方文档步骤已经很详细说明如何搭建了~

### 二、编译AAR

当我们用flutter实现好某一功能，希望开发android的小伙伴无需搭建flutter环境并如同依赖android library一样简单的接入使用，就需要编译打包成aar。

我们可以按照官网的操作：[将 Flutter module 作为依赖项](https://flutter.cn/docs/development/add-to-app/android/project-setup#add-the-flutter-module-as-a-dependency)

当我们把flutter模块做远程依赖时：

    在flutter模块目录下，执行：**flutter build aar** 。或者点击AS窗口的**Bulid->Flutter->Build AAR**

    命令执行成功后，会在模块根目录下生成一个build目录，在路径 **build/host/outputs/repo/com/example/xxx/xxxx/** 下找到 **flutter_release** 目录，里面会存放编译好的aar包

    同时还有一个pom的依赖说明文件，分别依赖了四个库：**flutter_embedding_release**(核心嵌入代码库)，以及各平台支持库**armeabi_v7a_release**、**arm64_v8a_release**、**x86_64_release**

*方案一*：把aar和所依赖的在线库资源都拷贝到对接此flutter模块的android library module下，在module中处理好所有的flutter对接，统一对外调用方法，把此module发布到自己的maven仓库，接入方只需要关注module库如何接入使用，而不需要关心flutter模块任何代码。

*方案二*：使用[com.kezong.fat-aar](https://github.com/kezong/fat-aar-android)插件，按照flutter官网的操作依赖方式写法。但需注意pom文件的依赖需要手动写一遍

```groovy
dependencies {
    ...
    embed 'com.example.xxx:flutter_release:1.0'
    embed 'io.flutter:flutter_embedding_release:1.0.0-6ba2af10bb05c88a2731482cedf2cfd11cf5af0b'
    embed 'io.flutter:armeabi_v7a_release:1.0.0-6ba2af10bb05c88a2731482cedf2cfd11cf5af0b'
    embed 'io.flutter:arm64_v8a_release:1.0.0-6ba2af10bb05c88a2731482cedf2cfd11cf5af0b'
//  embed 'io.flutter:x86_64_release:1.0.0-6ba2af10bb05c88a2731482cedf2cfd11cf5af0b' // 按需依赖
}
```

### 三、关于动态下发更新Flutter产物

未实践，参考：https://zhuanlan.zhihu.com/p/364616142


