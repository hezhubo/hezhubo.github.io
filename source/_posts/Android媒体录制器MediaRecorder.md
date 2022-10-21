---
title: Android媒体录制器MediaRecorder
date: 2022-08-24 10:55:01
tags: [音视频]
categories: 音视频
---

> 官方文档：https://developer.android.google.cn/reference/android/media/MediaRecorder

![MediaRecorder state diagram](https://developer.android.google.cn/static/images/mediarecorder_state_diagram.gif)

MediaRecorder支持捕获音频和视频并输出为音频文件或视频文件。如上图所示MediaRecorder的各种状态

### 初始化设置

* 实例化

  ```kotlin
  val mediaRecorder = MediaRecorder() // 已废弃
  val mediaRecorder = MediaRecorder(context) // recorder关联的上下文
  ```

  > 在搭载 Android 9（API 级别 28）或更高版本的设备上，在后台运行的应用将无法访问麦克风。因此，您的应用只在以下两种情况下才应录制音频：当其位于前台时，或者您在前台服务中添加了MediaRecorder实例时。[传送门](https://developer.android.google.cn/guide/topics/media/mediarecorder)

  由于高版本麦克风访问限制，因此需要传入Context

* 设置数据源

  ```kotlin
  mediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC)
  mediaRecorder.setVideoSource(MediaRecorder.VideoSource.SURFACE)
  ```

  由于我们不是开发系统级应用音频来源基本都是麦克风

  而视频来源有`MediaRecorder.VideoSource.CAMERA`和`MediaRecorder.VideoSource.SURFACE`，可录制摄像头数据或是surface数据

* 设置输出格式和输出路径

  ```kotlin
  mediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4) // 录制输出格式
  mediaRecorder.setOutputFile() // 设置输出文件可选file、path或是fileDescriptor
  ```

* 设置编码格式及编码参数

  ```kotlin
  mediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC) // 音频编码
  mediaRecorder.setAudioSamplingRate(44100) // 音频采样率
  mediaRecorder.setAudioEncodingBitRate(128000) // 音频码率
  mediaRecorder.setAudioChannels(1) // 声道数量
  mediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264) // 视频编码
  mediaRecorder.setVideoSize(1920, 1080) // 视频分辨率
  mediaRecorder.setVideoEncodingBitRate(5000_000) // 视频码率
  mediaRecorder.setVideoFrameRate(25) // 视频帧率（PS:录屏时，此值并没起到稳定帧率作用）
  ```

### 录制（主要操作方法）

```kotlin
mediaRecorder.prepare() // 在录制之前必须先执行prepare
mediaRecorder.start() // 开始录制
mediaRecorder.stop() // 停止录制
mediaRecorder.reset() // 重置为初始状态
mediaRecorder.release() // 释放资源

// 暂停和恢复录制，Android N（API 级别 24）或更高版本的设备上支持
mediaRecorder.pause()
mediaRecorder.resume()
```

### 摄像头录制示例

一般的操作步骤如下：

* 权限声明与申请
* 获取Camera实例，绑定到SurfaceView或TextureView进行预览
* 根据Camera相关参数，对MediaRecorder进行配置，注意相机方向、预览方向和录制的视频方向兼容问题
* 停止录制，释放MediaRecorder，最后释放Camera

```xml
<!-- AndroidManifest.xml中声明权限 -->
<!-- 摄像头相关权限 -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.FLASHLIGHT" />
<uses-feature android:name="android.hardware.camera" />
<uses-feature android:name="android.hardware.camera.autofocus" />
<!-- 录音权限 -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<!-- 写入存储权限 -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

```kotlin
// 示例代码
private fun initCamera() {
    if (!context.packageManager.hasSystemFeature(PackageManager.FEATURE_CAMERA)) {
        return // 没有摄像头
    }
    var mediaRecorder: MediaRecorder? = null
    var camera: Camera? = null
    val textureView = TextureView(context) // 获取TextureView实例
    textureView.surfaceTextureListener = object : TextureView.SurfaceTextureListener {
        override fun onSurfaceTextureAvailable(
            surface: SurfaceTexture,
            width: Int,
            height: Int
        ) {
            // 初始化摄像头
            try {
                camera = Camera.open(Camera.CameraInfo.CAMERA_FACING_FRONT)
                camera?.let {
                    it.setPreviewTexture(surface)
                    it.setDisplayOrientation(90) // 当前页面设置是竖屏，旋转90度
                    val parameters = it.parameters
                    val previewSizes =  parameters.supportedPreviewSizes // 获取支持的预览尺寸
                    val size = getMaxPreviewSize(previewSizes, width) // 根据自身实际情况选择合适的预览尺寸
                    parameters.setPreviewSize(size.width, size.height)
                    it.parameters = parameters
                    textureView.layoutParams.height = size.width
                    textureView.requestLayout()
                    it.startPreview()
                    it.autoFocus { success, camera ->
                        // 自动对焦
                    }
                    mediaRecorder = initMediaRecorder(it, size.width, size.height)
                    mediaRecorder?.start() // 开始录制
                }
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }

        /**
         * 获取最大的预览尺寸
         *
         * @param previewSizes 支持的尺寸集
         * @param maxHeight 最大高度限制
         */
        private fun getMaxPreviewSize(previewSizes: MutableList<Camera.Size>, maxHeight: Int) : Camera.Size {
            // 分辨率乘积从大到小排序
            previewSizes.sortWith(Comparator { o1, o2 ->
                return@Comparator if (o1.width * o1.height < o2.width * o2.height) {
                    1
                } else {
                    -1
                }
            })
            for (size in previewSizes) {
                if (size.height <= maxHeight) {
                    return size
                }
            }
            return previewSizes.last()
        }

        override fun onSurfaceTextureSizeChanged(
            surface: SurfaceTexture,
            width: Int,
            height: Int
        ) {
        }

        override fun onSurfaceTextureDestroyed(surface: SurfaceTexture): Boolean {
            // 停止录制并释放资源
            mediaRecorder?.let {
                it.stop()
                it.release()
            }
            mediaRecorder = null
            // 释放摄像头
            camera?.let {
                it.stopPreview()
                it.release()
            }
            camera = null
            return false
        }

        override fun onSurfaceTextureUpdated(surface: SurfaceTexture) {
        }
    }
}

private fun initMediaRecorder(camera: Camera, width: Int, height: Int) : MediaRecorder? {
    try {
        val mediaRecorder = MediaRecorder()
        // setCamera前必须调用camera.unlock()，且调用时机要在设置源之前
        camera.unlock()
        mediaRecorder.setCamera(camera)
        mediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC)
        mediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA)
        // 输出格式需要在设置编码器之前就设置好
        mediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
        // 自定义输出路径
        val outputPath = Environment.getExternalStorageDirectory().path + "/HezbRecorder/" + "${System.currentTimeMillis()}.mp4"
        mediaRecorder.setOutputFile(outputPath)
        // 音频相关参数
        mediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
        mediaRecorder.setAudioSamplingRate(44100)
        mediaRecorder.setAudioEncodingBitRate(128000)
        mediaRecorder.setAudioChannels(1)
        // 视频相关参数
        mediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.H264)
        // 使用预览尺寸为录制输出尺寸
        mediaRecorder.setVideoSize(width, height)
        // 通过CamcorderProfile获取对应视频质量的相关参数，注意若自定义设置的参数错误会导致录制出错
        val profile = CamcorderProfile.get(CamcorderProfile.QUALITY_HIGH)
        mediaRecorder.setVideoEncodingBitRate(profile.videoBitRate)
        mediaRecorder.setVideoFrameRate(profile.videoFrameRate)
        mediaRecorder.prepare()
        return mediaRecorder
    } catch (e: Exception) {
        return null
    }
}
```

### 屏幕录制示例

首先必须要申请录屏权限，拿到[MediaProjection](https://developer.android.google.cn/reference/android/media/projection/MediaProjection)实例

```kotlin
class MainActivity : Activity() {

    private var mMediaProjectionManager: MediaProjectionManager? = null

    /**
     * 请求录屏权限
     */
    private fun requestProjection(requestCode: Int) {
        mMediaProjectionManager =
            getSystemService(Context.MEDIA_PROJECTION_SERVICE) as MediaProjectionManager
        try {
            val captureIntent: Intent =
                mMediaProjectionManager?.createScreenCaptureIntent() ?: return
            startActivityForResult(captureIntent, requestCode)
        } catch (e: ActivityNotFoundException) {
        }
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent) {
        super.onActivityResult(requestCode, resultCode, data)
        if (resultCode == Activity.RESULT_OK) {
            val mediaProjection =
                mMediaProjectionManager?.getMediaProjection(resultCode, data) ?: return
            // 此处就得到了我们录制所需的MediaProjection实例
        }
    }

}
```

接着实例化MediaRecorder，并设置好相关配置

此处我们使用`MediaRecorder.VideoSource.SURFACE`作为数据源

```kotlin
val mediaRecorder = MediaRecorder()
mediaRecorder.setVideoSource(MediaRecorder.VideoSource.SURFACE)
...
mediaRecorder.prepare()
```

最后使用MediaRecorder的`Surface`作为创建[VirtualDisplay](https://developer.android.google.cn/reference/android/hardware/display/VirtualDisplay)的实例化参数，开启录制

```kotlin
try {
    val virtualDisplay = mediaProjection?.createVirtualDisplay(
        "Recorder-Display", // 虚拟显示器名称
        videoWidth,  // 虚拟显示器宽，一般使用录制输出的视频宽高
        videoHeight, // 虚拟显示器高
        dpi,				 // 虚拟显示器密度，必须大于0，一般传1即可
        DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC, // 虚拟显示器的标记（若窗口属性是FLAG_SECURE，此标记下窗口内容无法被录制）
        mediaRecorder.surface, // 虚拟现实器会把显示内容输出至此surface
        null, // 虚拟显示器的状态回调，一般无需关心
        null  // 为回调设置Handler，修改回调线程，默认是主线程回调，一般无需关心
    )
    mediaRecorder.start() // 开始录制
} catch (e: Exception) {
}

virtualDisplay.release() // 退出录制需释放VirtualDisplay
```

### 稳定屏幕录制帧率的方法

{% asset_img MediaRecorder_frame_rate_fix.jpg MediaRecorder_frame_rate_fix %}

通过OpenGL对Surface进行数据交换，创建定时渲染线程，固定时间间隔交换数据

