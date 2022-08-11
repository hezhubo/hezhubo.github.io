---
title: Android音频录制器AudioRecorder
date: 2022-07-25 11:29:03
tags: [音视频]
categories: 音视频
---

> 官方文档：https://developer.android.google.cn/reference/android/media/AudioRecord

### 权限

```xml
<!-- AndroidManifest.xml中声明权限 -->
<!-- 录音权限 -->
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

```kotlin
/**
 * 校验是否有权限，若无权限则去申请权限
 */
private fun checkPermission() : Boolean {
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    val recordAudio = ActivityCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO)
    if (recordAudio != PackageManager.PERMISSION_GRANTED) {
      ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.RECORD_AUDIO), 123)
    	return false
    }
  }
  return true
}

override fun onRequestPermissionsResult(
  requestCode: Int,
  permissions: Array<out String>,
  grantResults: IntArray
) {
  super.onRequestPermissionsResult(requestCode, permissions, grantResults)
  if (requestCode == 123) {
    for ((index, result) in grantResults.withIndex()) {
      if (result != PackageManager.PERMISSION_GRANTED) {
        // permissions[index] 此权限授权失败
        return
      }
    }
  }
}
```

### 实例化

```kotlin
private fun initAudioRecord(): Boolean {
  val audioSource = MediaRecorder.AudioSource.MIC // 音频数据源，当前为麦克风
  val sampleRateInHz = 44100 // 音频采样率
  val channelConfig = 1 // 音频通道数
  val audioFormat = AudioFormat.ENCODING_PCM_16BIT // 音频编码格式
  // 获取最小的缓冲区字节大小
  val bufferSizeInBytes = AudioRecord.getMinBufferSize(
    sampleRateInHz,
    channelConfig,
    audioFormat
  )
  try {
    mAudioRecord = AudioRecord(
      audioSource,
      sampleRateInHz,
      channelConfig,
      audioFormat,
      bufferSizeInBytes
    )
  } catch (e: Exception) {
    return false
  }

  if (AudioRecord.STATE_INITIALIZED != mAudioRecord?.state) {
    // 校验当前状态，确保AudioRecord正常被初始化
    return false
  }
  return true
}
```

### 录制（主要操作方法）

```kotlin
// 开始录音
mAudioRecord?.startRecording()

// 停止录音
mAudioRecord?.stop()

// 释放资源
mAudioRecord?.release()

// 获取当前录制状态 RECORDSTATE_STOPPED:录制停止, RECORDSTATE_RECORDING:录制中
val recordingState = mAudioRecord?.recordingState
```

###读取(保存)录制数据

```kotlin
/**
 * 录音线程
 * 我们需要新建一个线程来不停的获取录制的音频数据
 */
private inner class AudioRecordThread : Thread() {
  private var isRunning = true

  fun quit() {
    isRunning = false
  }

  override fun run() {
    val audioBuffer = ByteArray(bufferSizeInBytes) // bufferSizeInBytes是构建AudioRecord设置的缓冲区字节大小参数
    while (isRunning) {
      val size = mAudioRecord?.read(audioBuffer, 0, bufferSizeInBytes) ?: 0
      if (isRunning && size > 0) {
        // 此时就得到了PCM格式的音频数据，在 audioBuffer 中 0-size
        // 可以对数据进行加工（变声等）处理
      }
    }
  }
}
```

### 录制系统声音

Android 10开始，支持录制系统正在播放的声音。官方说明：https://developer.android.google.cn/guide/topics/media/av-capture

除了要声明并获取录音权限外，还需要申请如录屏一样的权限[`MediaProjectionManager.createScreenCaptureIntent()`](https://developer.android.google.cn/reference/android/media/projection/MediaProjectionManager#createScreenCaptureIntent())

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

创建AudioRecord实例

```kotlin
private fun initPlaybackAudioRecord(mediaProjection: MediaProjection): Boolean {
  val sampleRateInHz = 44100 // 音频采样率
  val channelConfig = 1 // 音频通道数
  val audioFormat = AudioFormat.ENCODING_PCM_16BIT // 音频编码格式
  // 获取最小的缓冲区字节大小
  val bufferSizeInBytes = AudioRecord.getMinBufferSize(
    sampleRateInHz,
    channelConfig,
    audioFormat
  )
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    val apccBuilder = AudioPlaybackCaptureConfiguration.Builder(mediaProjection)
    apccBuilder.addMatchingUsage(AudioAttributes.USAGE_UNKNOWN)
    apccBuilder.addMatchingUsage(AudioAttributes.USAGE_GAME) // 游戏类型声音
    apccBuilder.addMatchingUsage(AudioAttributes.USAGE_MEDIA) // 媒体类型声音
    val audioRecorderBuilder = AudioRecord.Builder()
//  .setAudioSource(MediaRecorder.AudioSource.DEFAULT) // 注意！无法同时设置音频来源
    .setAudioFormat(
      AudioFormat.Builder()
      .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
      .setSampleRate(sampleRateInHz)
      .setChannelMask(channelConfig)
      .build()
    )
    .setBufferSizeInBytes(bufferSizeInBytes)
    .setAudioPlaybackCaptureConfig(apccBuilder.build())
    mPlaybackAudioRecord = try {
      audioRecorderBuilder.build()
    } catch (e: Exception) {
      null
    }
    if (AudioRecord.STATE_INITIALIZED != mPlaybackAudioRecord?.state) {
      return false
    }
    return true
  }
  return false
}
```

指定录制系统声音类型：[AudioPlaybackCaptureConfiguration.addMatchingUsage()](https://developer.android.google.cn/reference/android/media/AudioPlaybackCaptureConfiguration.addMatchingUsage(int)) 

指定不去录制系统声音类型：[AudioPlaybackCaptureConfiguration.excludeUsage()](https://developer.android.google.cn/reference/android/media/AudioPlaybackCaptureConfiguration.excludeUsage(int)) 

指定录制某UID(app)的声音：[AudioPlaybackCaptureConfiguration.addMatchingUid()](https://developer.android.google.cn/reference/android/media/AudioPlaybackCaptureConfiguration.addMatchingUid(int)) 

指定不录制某UID(app)的声音：[AudioPlaybackCaptureConfiguration.excludeUid()](https://developer.android.google.cn/reference/android/media/AudioPlaybackCaptureConfiguration.excludeUid(int)) 

注意：**addMatchingUsage()** 和 **excludeUsage()** 不能同时调用，**addMatchingUid()** 和 **excludeUid()** 也是不能同时调用

最后，并不是所有系统输出的声音都可以被捕获录制到

当app的target api小于Android 10 (API level 29)，默认是不能被录制其声音的，需要在manifest的application节点配置**android:allowAudioPlaybackCapture="true"**，才能被录制到

而target api在Android 10 (API level 29)以上默认就是允许被录制其声音

更具体详细的录制限制，请参考官方文档。