---
title: android多媒体播放器之MediaPlayer
date: 2016-08-27 20:24:15
tags: [mediaplayer,多媒体,音视频]
categories: Android
---

**废话少说，先上一张无比重要的图：MediaPlayer状态图，读懂此图那后面的都是废话了。**
![android官方MediaPlayer状态图](https://developer.android.google.cn/images/mediaplayer_state_diagram.gif?hl=zh-cn)

#### MediaPlayer的简单使用
```java
MediaPlayer mediaPlayer = new MediaPlayer();
try {
    mediaPlayer.setDataSource("/mnt/sdcard/test.mp3"); // 设置多媒体文件路径，支持本地文件，http，rtsp协议
    mediaPlayer.prepare(); // 以同步(阻塞)方式，预处理（加载）多媒体文件，此方法一般用于本地文件 
    mediaPlayer.start(); // 开始播放
} catch (IOException e) {
    e.printStackTrace();
}
```
#### MediaPlayer的基本方法说明
##### 创建MediaPlayer对象
```java
// 基本的构造方法
MediaPlayer mediaPlayer = new MediaPlayer();

// 静态的构造方法
Context context; // 上下文
Uri uri; // 多媒体文件的URI
SurfaceHolder holder; // surface持有者，用于视频画面输出
AudioAttributes audioAttributes; // 封装音频流信息的属性集合类
int audioSessionId; // 音频的Session Id
int resid; // 多媒体资源Id
MediaPlayer mediaPlayer = MediaPlayer.create(context, uri);
MediaPlayer mediaPlayer = MediaPlayer.create(context, uri, holder);
MediaPlayer mediaPlayer = MediaPlayer.create(context, uri, holder, audioAttributes, audioSessionId);
MediaPlayer mediaPlayer = MediaPlayer.create(context, resid);
MediaPlayer mediaPlayer = MediaPlayer.create(context, resid, audioAttributes, audioSessionId);
```

##### 设置MediaPlayer的各种监听器
```java
// 预处理监听器
mediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
    @Override
    public void onPrepared(MediaPlayer mp) {
        // 在MediaPlayer完成加载多媒体文件时回调此方法
        // 此时可以获取多媒体文件信息（如视频宽高，时长等）
        // 可以开始执行播放
    }
});
// 播放完成监听器
mediaPlayer.setOnCompletionListener(new MediaPlayer.OnCompletionListener() {
    @Override
    public void onCompletion(MediaPlayer mp) {
        // 在MediaPlayer播放完成时回调此方法
    }
});
// 播放出错监听器
mediaPlayer.setOnErrorListener(new MediaPlayer.OnErrorListener() {
    @Override
    public boolean onError(MediaPlayer mp, int what, int extra) {
        // 使用播放器过程中出现错误会回调此方法
        // what 发生的错误类型, extra 具体的错误代码
        return false;
    }
});
// 播放过程中部分信息监听器
mediaPlayer.setOnInfoListener(new MediaPlayer.OnInfoListener() {
    @Override
    public boolean onInfo(MediaPlayer mp, int what, int extra) {
        // 在播放器播放过程中部分状态信息会回调此方法
        // what 信息或警告的类型, extra 具体的信息代码
        // 如 what: MEDIA_INFO_BUFFERING_START 开始缓冲
        // MEDIA_INFO_BUFFERING_END 缓冲完成
        return false;
    }
});
// 视频尺寸变化监听器
mediaPlayer.setOnVideoSizeChangedListener(new MediaPlayer.OnVideoSizeChangedListener() {
    @Override
    public void onVideoSizeChanged(MediaPlayer mp, int width, int height) {
        // 在播放视频时视频宽高发生变化回调此方法
        // 此时一般会去处理视频输出的View的宽高布局
    }
});
// 缓冲进度更新监听器
mediaPlayer.setOnBufferingUpdateListener(new MediaPlayer.OnBufferingUpdateListener() {
    @Override
    public void onBufferingUpdate(MediaPlayer mp, int percent) {
        // 在缓冲进度改变时会回调此方法
        // percent 缓冲百分比
        // 一般用于更新进度条的第二进度
    }
});
// 播放跳转完成监听器
mediaPlayer.setOnSeekCompleteListener(new MediaPlayer.OnSeekCompleteListener() {
    @Override
    public void onSeekComplete(MediaPlayer mp) {
        // 在调用seekTo()，跳转完成后会回调此方法
    }
});
// 字幕监听器
mediaPlayer.setOnTimedTextListener(new MediaPlayer.OnTimedTextListener() {
    @Override
    public void onTimedText(MediaPlayer mp, TimedText text) {
        // 当有字幕需要显示时会回调此方法
        // text 包含字幕内容，格式
        // 我暂时没有用过此监听器
    }
});
// API23 媒体数据可用监听器
mediaPlayer.setOnTimedMetaDataAvailableListener(new MediaPlayer.OnTimedMetaDataAvailableListener() {
    @Override
    public void onTimedMetaDataAvailable(MediaPlayer mp, TimedMetaData data) {
        // 当有可用的元数据时回调此方法
        // data 数据元，包含时间戳
        // 我暂时没有用过此监听器，我猜想可以用此回调来实现边看边缓存视频
    }
});
```

##### MediaPlayer的操作方法
###### 设置数据元
```java
Context context; // 上下文
Uri uri; // 多媒体文件的URI
Map<String, String> headers; // http的请求头
String path; // 多媒体路径，支持本地文件，http，rtsp协议
FileDescriptor fd; // 多媒体文件描述符，可通过AssetManager去构建
long offset; // 以字节为单位的数据偏移量
long length; // 要播放的数据字节长度
MediaDataSource dataSource; // 为framework层提供媒体数据的类
mediaPlayer.setDataSource(context, uri);
mediaPlayer.setDataSource(context, uri, headers); // 暂时没用过
mediaPlayer.setDataSource(path);
mediaPlayer.setDataSource(fd);
mediaPlayer.setDataSource(fd, offset, length); // 暂时没用过
mediaPlayer.setDataSource(dataSource); // 暂时没用过
```
###### 预处理(加载)多媒体数据
```java
mediaPlayer.prepare(); // 会阻塞当前线程
mediaPlayer.prepareAsync(); // 异步加载
```
###### 开始播放
```java
mediaPlayer.start(); // 需要在预处理完成后才会起作用
```
###### 暂停播放
```java
mediaPlayer.pause(); // 需要在预处理完成后才会起作用
```
###### 跳转到指定位置
```java
int msec; // 跳转位置（单位毫秒） 取值要大于0，小于视频总时长
mediaPlayer.seekTo(msec); // 需要在预处理完成后才会起作用
```
###### 获取当前播放位置
```java
int currentPosition = mediaPlayer.getCurrentPosition(); // 需要在预处理完成后才会起作用
```
###### 获取当前总时长
```java
int duration = mediaPlayer.getDuration(); // 需要在预处理完成后才会起作用
```
###### 停止播放
```java
mediaPlayer.stop(); // 需要在预处理完成后才会起作用，若想继续播放，需要重新加载多媒体文件
```
###### 重置播放器
```java
mediaPlayer.reset(); // 重置mediaPlayer为最初状态，再次使用需要重新设置多媒体路径
```
###### 释放资源
```java
mediaPlayer.release(); // 释放与mediaPlayer对象关联的所有资源
```
###### 设置视频画面输出
```java
// 通过SurfaceView来显示视频
SurfaceView surfaceView; // 通过xml资源或是使用代码自行创建
// 需要给surface持有者添加监听器
surfaceView.getHolder().addCallback(new SurfaceHolder.Callback() {
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        // surfaceView成功创建，此时holder才可用
        mediaPlayer.setDisplay(holder);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        // surface发生改变时
        // format 格式， width 宽， height 高
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        // surfaceView销毁时
    }
});

// 通过TextureView来显示视频
TextureView textureView; // 通过xml资源或是使用代码自行创建
textureView.setSurfaceTextureListener(new TextureView.SurfaceTextureListener() {
    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        // 当SurfaceTexture可用时
        // width 纹理宽， height 纹理高
        Surface displaySurface = new Surface(surface); // 通过SurfaceTexture来构建Surface 
        mediaPlayer.setSurface(displaySurface);
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int width, int height) {
        // 纹理尺寸发生改变时
    }

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
        // textureView销毁时
        return false;
    }

    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {
        // textureView重新设置了SurfaceTexture
    }
});
```
###### 获取视频宽高
```java
int width = mediaPlayer.getVideoWidth(); // 需要在预处理完成后才会起作用
int height = mediaPlayer.getVideoHeight(); // 需要在预处理完成后才会起作用
```
其他方法就不一一列出了，可以自行查看[MediaPlayer源码](https://developer.android.google.cn/reference/android/media/MediaPlayer)

