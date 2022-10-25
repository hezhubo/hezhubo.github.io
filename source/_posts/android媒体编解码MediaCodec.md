---
title: android媒体编解码MediaCodec
date: 2022-10-21 10:28:15
tags: [音视频]
categories: 音视频
---

> 官方文档：https://developer.android.google.cn/reference/android/media/MediaCodec
>
> https://developer.android.google.cn/guide/topics/media/media-codecs

MediaCodec，Android基础的编解码器

### 初始化设置

```kotlin
// 通过编解码器的名称创建
val mediaCodec = MediaCode.createByCodecName("OMX.google.h264.encoder")

// 通过mime type创建编码器
val encoderCodec = MediaCode.createEncoderByType(MediaFormat.MIMETYPE_VIDEO_AVC)
// 通过mime type创建解码器
val decoderCodec = MediaCode.createDecoderByType(MediaFormat.MIMETYPE_VIDEO_AVC)
```

codec name可以通过[**MediaCodecList**](https://developer.android.google.cn/reference/android/media/MediaCodecList)获取所有支持的编解码器，从而取得编解码器名称

```kotlin
val codecList = MediaCodecList(MediaCodecList.ALL_CODECS)
// 获取所有编解码器
val mediaCodecInfoList = codecList.codecInfos
for (mediaCodecInfo in mediaCodecInfoList) {
  	val codecName = mediaCodecInfo.name
}

// 直接通过MediaFormat获取编解码器名称
val encoderCodecName = codecList.findEncoderForFormat(mediaFormat)
val decoderCodecName = codecList.findDecoderForFormat(mediaFormat)
```

[**MediaCodecInfo**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo)存放着编码器相关信息

```kotlin
// 是否仅支持软编解码
mediaCodecInfo.isSoftwareOnly

// 是否支持硬件加速
mediaCodecInfo.isHardwareAccelerated

// 是否为编码器，false就是解码器
mediaCodecInfo.isEncoder

// 获取编解码器所支持的mime type
mediaCodecInfo.getSupportedTypes

// 通过mime type获取编解码器组件的功能
val codecCapabilities = mediaCodecInfo.getCapabilitiesForType(mimeType)
// 如果当前不是音频的编解码器，返回null
val audioCapabilities = codecCapabilities.audioCapabilities
// 如果当前不是视频的编解码器，返回null
val videoCapabilities = codecCapabilities.videoCapabilities
```

接下来我们来看MediaCodecInfo的内部类

* [**MediaCodecInfo.CodecCapabilities**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo.CodecCapabilities) 编解码器的功能封装(profile/level、colorspaces...)

* [**MediaCodecInfo.AudioCapabilities**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo.AudioCapabilities) 音频编解码器支持的配置功能(码率、采样率、通道数...)

* [**MediaCodecInfo.VideoCapabilities**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo.VideoCapabilities) 视频编解码器支持的配置功能(分辨率、码率、帧率...)

* [**CodecProfileLevel**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo.CodecProfileLevel) 编解码器配置封装(列出支持的编解码格式配置)

  ```kotlin
  // 编解码器支持的配置数组，不同配置编码会限制是否支持B帧、编码算法等
  val codecProfileLevels = codecCapabilities.profileLevels
  ```

* [**EncoderCapabilities**](https://developer.android.google.cn/reference/android/media/MediaCodecInfo.EncoderCapabilities) 编码器支持的功能(主要是编码模式)

  * *BITRATE_MODE_CQ* 恒定质量模式
  * *BITRATE_MODE_VBR* 可变码率模式
  * *BITRATE_MODE_CBR* 恒定码率模式
  * *BITRATE_MODE_CBR_FD* 带帧下降的恒定比特率模式

  ```java
  // 支持的编码模式，framework代码写死了仅支持VBR，isBitrateModeSupported(int mode)返回支持的编码模式
  private static final Feature[] bitrates = new Feature[] {
    new Feature("VBR", BITRATE_MODE_VBR, true),
    new Feature("CBR", BITRATE_MODE_CBR, false),
    new Feature("CQ",  BITRATE_MODE_CQ,  false),
    new Feature("CBR-FD", BITRATE_MODE_CBR_FD, false)
  };
  ```

通过编解码器的信息我们知道了所选的编解码器可支持的属性配置，下面使用[**MediaFormat**](https://developer.android.google.cn/reference/android/media/MediaFormat)进行配置

```kotlin
// 视频编码例子
val encoderCodec = MediaCode.createEncoderByType(MediaFormat.MIMETYPE_VIDEO_AVC)
val encodeFormat = MediaFormat().apply {
  	setString(MediaFormat.KEY_MIME, mineType)
  	setInteger(MediaFormat.KEY_WIDTH, videoWidth)
  	setInteger(MediaFormat.KEY_HEIGHT, videoHeight)
  	... // 具体查看源码所有支持的配置key
}
encoderCodec.configure(encodeFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE)
```

###  视频编码输入Surface

**MediaCodec.createInputSurface()** 创建*surface*用作编码器的输入，代替输入缓冲区。把*surface*作为创建 **VirtualDisplay** 的参数即可录制虚拟屏幕的画面

```kotlin
val surface = encoderCodec.createInputSurface()
val virtualDisplay = mediaProjection.createVirtualDisplay(
    "Recorder-Display",
    videoWidth,
    videoHeight,
    dpi,
    DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC,
    surface,
    null,
    null
)
```

由于屏幕刷新率fps在不同的界面(应用)会不一致，一般在手机桌面时刷新率极低，到某些游戏中刷新率就非常高，如上设置录制方式进行录制会存在录制的视频**帧率不稳定**的问题，因此我们一般不直接使用MediaCodec创建的Surface来创建VirtualDisplay，而是通过OpenGL对Surface进行数据交换(利用OpenGL一种离屏渲染方式FBO实现surface间纹理渲染)，创建定时渲染线程，固定时间间隔交换数据

{% asset_img OpenGL_FBO.jpg OpenGL_FBO %}

* 初始化EGL

  ```kotlin
  // OpenGL 示例关键代码
  // 打开与EGL显示服务器的连接
  val eglDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)
  // 初始化EGL
  val version = IntArray(2)
  if (!EGL14.eglInitialize(eglDisplay, version, 0, version, 1)) {
    	throw RuntimeException("unable to initialize EGL14")
  }
  // 让EGL选择匹配的EGLConfig
  val attribList = intArrayOf(
      EGL14.EGL_RED_SIZE, 8,     // R
      EGL14.EGL_GREEN_SIZE, 8,   // G
      EGL14.EGL_BLUE_SIZE, 8,    // B
      EGL14.EGL_ALPHA_SIZE, 8,   // A
      EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT, // egl版本 2.0
      EGLExt.EGL_RECORDABLE_ANDROID, 1, // android特定配置
      EGL14.EGL_NONE // 结束符
  )
  val configs = arrayOfNulls<EGLConfig>(1)
  val numConfigs = IntArray(1)
  val success = EGL14.eglChooseConfig(eglDisplay, attribList, 0, configs, 0, 1, numConfigs, 0)
  if (!success || numConfigs[0] <= 0) {
    	throw RuntimeException("unable to find RGBA8888 / 2 EGLConfig!")
  }
  val eglConfig = configs[0]
  // 创建渲染上下文
  val contextAttribList = intArrayOf(
      EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
      EGL14.EGL_NONE
  )
  val eglContext = EGL14.eglCreateContext(eglDisplay, eglConfig, EGL14.EGL_NO_CONTEXT, contextAttribList, 0)
  
  // 通过MediaCodec创建的Surface来创建EGL渲染窗口 (surface = encoderCodec.createInputSurface())
  val eglSurface = EGL14.eglCreateWindowSurface(eglDisplay, eglConfig, surface, intArrayOf(EGL14.EGL_NONE), 0)
  ```

* 创建绘制相关的纹理及其绑定的帧缓冲区对象

  ```kotlin
  // OpenGL 示例关键代码
  // 生成一个纹理id
  val textures = IntArray(1)
  GLES20.glGenTextures(1, textures, 0)
  val screenTextureId = textures[0]
  // 将图像流中的帧捕获为OpenGL ES纹理
  // 创建OpenGL ES纹理
  val screenSurfaceTexture = SurfaceTexture(screenTextureId).apply {
      setDefaultBufferSize(videoWidth, videoHeight) // 修改默认缓存大小
      setOnFrameAvailableListener {
        	// 帧可用，通知渲染
        	updateTexImage()
       		// draw frame
      }
  }
  val virtualDisplay = mediaProjection.createVirtualDisplay(
      "Recorder-Display",
      videoWidth,
      videoHeight,
      dpi,
      DisplayManager.VIRTUAL_DISPLAY_FLAG_PUBLIC,
      Surface(screenSurfaceTexture), // 创建纹理的surface，作为VirtualDisplay屏幕数据原始缓冲区
      null,
      null
  )
  
  // 渲染线程需进行初始化
  // 创建空纹理及帧缓冲区对象，用于接收屏幕数据
  val sourceTexture = GlUtil.createImageTexture(videoWidth, videoHeight)
  val sourceFramebuffer = GlUtil.createFramebufferLinkTexture2D(sourceTexture)
   // 开启 GL_TEXTURE_EXTERNAL_OES 纹理
  GLES20.glEnable(GLES11Ext.GL_TEXTURE_EXTERNAL_OES)
  // 创建纹理程序
  val drawProgram = Texture2DProgram(fragmentShaderCode = Texture2DProgram.FRAGMENT_SHADER_2D)
  val source2dProgram = Texture2DProgram(fragmentShaderCode = Texture2DProgram.FRAGMENT_SHADER_SOURCE2D)
  // 顶点缓冲区
  val drawIndicesBuffer = GlUtil.getDrawIndicesBuffer()
  val shapeVerticesBuffer = GlUtil.getShapeVerticesBuffer()
  val source2DVerticesBuffer = GlUtil.getScreenTextureVerticesBuffer()
  val targetVerticesBuffer = GlUtil.getTargetTextureVerticesBuffer()
  
  
  ```

* 渲染屏幕数据

  ```kotlin
  // OpenGL 示例关键代码
  // 在SurfaceTexture的onFrameAvailable回调里，执行绘制帧
  // 指定上下文
  EGL14.eglMakeCurrent(eglDisplay, eglSurface, eglSurface, eglContext)
  // 绘制帧缓冲区附着的2D纹理
  GlUtil.draw2DFramebuffer(
      GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
      screenTextureId, // virtualDisplay输出的纹理id
      sourceFramebuffer, // sourceTexture绑定的帧缓冲区
      videoWidth,
      videoHeight,
      source2dProgram,
      shapeVerticesBuffer,
      source2DVerticesBuffer,
      drawIndicesBuffer
  )
  ```

* 定时渲染EGL窗口，数据写入编码器Surface

  ```kotlin
  // OpenGL 示例关键代码
  // 绘制2D纹理
  GlUtil.drawTexture2D(
      sourceTexture, // 最终绘制到的帧缓冲区所附着的2D纹理
      drawProgram,
      shapeVerticesBuffer,
      targetVerticesBuffer,
      drawIndicesBuffer
  )
  // 设置pts
  EGLExt.eglPresentationTimeANDROID(eglDisplay, eglSurface, nanoseconds) // nanoseconds时间戳，单位纳秒
  // Display与Surface数据交换
  EGL14.eglSwapBuffers(eglDisplay, eglSurface)
  ```

**GlUtil** 工具类代码

```kotlin
package com.hezb.live.recorder.gles

import android.graphics.Bitmap
import android.opengl.*
import com.hezb.live.recorder.util.LogUtil
import java.nio.ByteBuffer
import java.nio.ByteOrder
import java.nio.FloatBuffer
import java.nio.ShortBuffer

/**
 * Project Name: AndroidScreenLive
 * File Name:    GlUtil
 *
 * Description: OpenGL工具类.
 *
 * @author  hezhubo
 * @date    2022年07月19日 21:15
 */
object GlUtil {

    /** 无效纹理id */
    const val NO_TEXTURE = -1

    private const val SHORT_SIZE_BYTES = 2 // short占2字节
    private const val FLOAT_SIZE_BYTES = 4 // float占4字节

    /** 绘制的(两个)三角形顶点序号标记 */
    private val drawIndices = shortArrayOf(0, 1, 2, 0, 2, 3)

    /** 四边形(共用一边的两个三角形)顶点坐标 (gl_Position) */
    private val squareVertices = floatArrayOf(
        -1.0f, 1.0f,
        -1.0f, -1.0f,
        1.0f, -1.0f,
        1.0f, 1.0f
    )

    /** 屏幕纹理顶点坐标 (vTextureCoord) */
    private val screenTextureVertices = floatArrayOf(
        0.0f, 0.0f,
        0.0f, 1.0f,
        1.0f, 1.0f,
        1.0f, 0.0f
    )

    /** 绘制输出纹理顶点坐标 */
    private val targetTextureVertices = floatArrayOf(
        0.0f, 1.0f,
        0.0f, 0.0f,
        1.0f, 0.0f,
        1.0f, 1.0f
    )

    fun getDrawIndicesBuffer(): ShortBuffer {
        // 创建内存块缓冲区
        val buffer = ByteBuffer
            .allocateDirect(SHORT_SIZE_BYTES * drawIndices.size)
            .order(ByteOrder.nativeOrder())
            .asShortBuffer()
        buffer.put(drawIndices)
        buffer.position(0)
        return buffer
    }

    fun getShapeVerticesBuffer(): FloatBuffer {
        val buffer = ByteBuffer
            .allocateDirect(FLOAT_SIZE_BYTES * squareVertices.size)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
        buffer.put(squareVertices)
        buffer.position(0)
        return buffer
    }

    fun getScreenTextureVerticesBuffer(): FloatBuffer {
        val buffer = ByteBuffer
            .allocateDirect(FLOAT_SIZE_BYTES * screenTextureVertices.size)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
        buffer.put(screenTextureVertices)
        buffer.position(0)
        return buffer
    }

    fun getTargetTextureVerticesBuffer(): FloatBuffer {
        val buffer = ByteBuffer
            .allocateDirect(FLOAT_SIZE_BYTES * targetTextureVertices.size)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
        buffer.put(targetTextureVertices)
        buffer.position(0)
        return buffer
    }

    /**
     * 创建一个指定大小的空纹理对象
     *
     * @param width
     * @param height
     * @param format
     * @return 纹理id
     */
    fun createImageTexture(width: Int, height: Int, format: Int = GLES20.GL_RGBA): Int {
        // 创建一个纹理对象
        val textures = IntArray(1)
        GLES20.glGenTextures(1, textures, 0)
        // 绑定纹理对象
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textures[0])
        // 加载2D纹理（此处为空纹理）
        GLES20.glTexImage2D(
            GLES20.GL_TEXTURE_2D,
            0,
            format,
            width,
            height,
            0,
            format,
            GLES20.GL_UNSIGNED_BYTE,
            null // 包含图像的实际像素数据
        )
        // 设置纹理过滤模式
        GLES20.glTexParameterf(
            GLES20.GL_TEXTURE_2D,
            GLES20.GL_TEXTURE_MIN_FILTER,
            GLES20.GL_LINEAR.toFloat()
        )
        GLES20.glTexParameterf(
            GLES20.GL_TEXTURE_2D,
            GLES20.GL_TEXTURE_MAG_FILTER,
            GLES20.GL_LINEAR.toFloat()
        )
        GLES20.glTexParameterf(
            GLES20.GL_TEXTURE_2D,
            GLES20.GL_TEXTURE_WRAP_S,
            GLES20.GL_CLAMP_TO_EDGE.toFloat()
        )
        GLES20.glTexParameterf(
            GLES20.GL_TEXTURE_2D,
            GLES20.GL_TEXTURE_WRAP_T,
            GLES20.GL_CLAMP_TO_EDGE.toFloat()
        )

        // 取消绑定
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0)

        return textures[0]
    }

    /**
     * 创建一个帧缓冲区对象，并连接到一个2D纹理
     *
     * @param textureId
     * @return 帧缓冲区id
     */
    fun createFramebufferLinkTexture2D(textureId: Int): Int {
        // 分配一个帧缓冲区对象（FOB）
        val framebuffer = IntArray(1)
        GLES20.glGenFramebuffers(1, framebuffer, 0)
        // 设置当前帧缓冲区对象
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, framebuffer[0])
        // 连接一个2D纹理作为帧缓冲区附着
        GLES20.glFramebufferTexture2D(
            GLES20.GL_FRAMEBUFFER,
            GLES20.GL_COLOR_ATTACHMENT0,
            GLES20.GL_TEXTURE_2D,
            textureId,
            0
        )

        // 取消绑定
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0)

        return framebuffer[0]
    }

    /**
     * 创建OpenGL着色器程序
     *
     * @param vertexShaderCode 顶点着色器代码
     * @param fragmentShaderCode 片段着色器代码
     * @return 程序(指针)，返回0则发生错误
     */
    fun createProgram(vertexShaderCode: String, fragmentShaderCode: String): Int {
        val vertexShader = GLES20.glCreateShader(GLES20.GL_VERTEX_SHADER)
        val fragmentShader = GLES20.glCreateShader(GLES20.GL_FRAGMENT_SHADER)
        GLES20.glShaderSource(vertexShader, vertexShaderCode)
        GLES20.glShaderSource(fragmentShader, fragmentShaderCode)
        val status = IntArray(1)
        GLES20.glCompileShader(vertexShader)
        GLES20.glGetShaderiv(vertexShader, GLES20.GL_COMPILE_STATUS, status, 0)
        if (GLES20.GL_FALSE == status[0]) {
            LogUtil.e(msg = "vertex shader compile, failed : ${GLES20.glGetShaderInfoLog(vertexShader)}")
            return 0
        }
        GLES20.glCompileShader(fragmentShader)
        GLES20.glGetShaderiv(fragmentShader, GLES20.GL_COMPILE_STATUS, status, 0)
        if (GLES20.GL_FALSE == status[0]) {
            LogUtil.e(msg = "fragment shader compile, failed : ${GLES20.glGetShaderInfoLog(fragmentShader)}")
            return 0
        }
        val program = GLES20.glCreateProgram()
        GLES20.glAttachShader(program, vertexShader)
        GLES20.glAttachShader(program, fragmentShader)
        GLES20.glLinkProgram(program)
        GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, status, 0)
        if (GLES20.GL_FALSE == status[0]) {
            LogUtil.e(msg = "link program, failed : ${GLES20.glGetProgramInfoLog(program)}")
            return 0
        }
        return program
    }

    /**
     * 绘制帧缓冲区附着的2D纹理
     *
     * @param textureTarget
     * @param textureId
     * @param framebufferId
     * @param width
     * @param height
     * @param texture2DProgram
     * @param shapeVerticesBuffer
     * @param textureVerticesBuffer
     * @param drawIndicesBuffer
     */
    fun draw2DFramebuffer(
        textureTarget: Int,
        textureId: Int,
        framebufferId: Int,
        width: Int,
        height: Int,
        texture2DProgram: Texture2DProgram,
        shapeVerticesBuffer: FloatBuffer,
        textureVerticesBuffer: FloatBuffer,
        drawIndicesBuffer: ShortBuffer
    ) {
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, framebufferId)
        GLES20.glUseProgram(texture2DProgram.program)
        // 激活纹理单元
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0)
        GLES20.glBindTexture(textureTarget, textureId)
        GLES20.glUniform1i(texture2DProgram.uTextureLoc, 0)
        GLES20.glEnableVertexAttribArray(texture2DProgram.aPositionLoc)
        GLES20.glEnableVertexAttribArray(texture2DProgram.aTextureCoordLoc)
        // 指定顶点数组
        GLES20.glVertexAttribPointer(
            texture2DProgram.aPositionLoc,
            2,
            GLES20.GL_FLOAT,
            false,
            0,
            shapeVerticesBuffer
        )
        GLES20.glVertexAttribPointer(
            texture2DProgram.aTextureCoordLoc,
            2,
            GLES20.GL_FLOAT,
            false,
            0,
            textureVerticesBuffer
        )
        // 指定视口尺寸
        GLES20.glViewport(0, 0, width, height)
        // 清除颜色缓冲区
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f)
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT)
        // 从数组中获得数据渲染图元
        GLES20.glDrawElements(
            GLES20.GL_TRIANGLES,
            drawIndicesBuffer.limit(),
            GLES20.GL_UNSIGNED_SHORT,
            drawIndicesBuffer
        )
        // 向图形硬件提交缓冲区里的指令
        GLES20.glFinish()
        // 解绑一系列
        GLES20.glDisableVertexAttribArray(texture2DProgram.aPositionLoc)
        GLES20.glDisableVertexAttribArray(texture2DProgram.aTextureCoordLoc)
        GLES20.glBindTexture(textureTarget, 0)
        GLES20.glUseProgram(0)
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0)
    }

    /**
     * 绘制2D纹理
     *
     * @param textureId
     * @param texture2DProgram
     * @param shapeVerticesBuffer
     * @param textureVerticesBuffer
     * @param drawIndicesBuffer
     */
    fun drawTexture2D(
        textureId: Int,
        texture2DProgram: Texture2DProgram,
        shapeVerticesBuffer: FloatBuffer,
        textureVerticesBuffer: FloatBuffer,
        drawIndicesBuffer: ShortBuffer
    ) {
        GLES20.glUseProgram(texture2DProgram.program)
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0)
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId)
        GLES20.glUniform1i(texture2DProgram.uTextureLoc, 0)
        GLES20.glEnableVertexAttribArray(texture2DProgram.aPositionLoc)
        GLES20.glEnableVertexAttribArray(texture2DProgram.aTextureCoordLoc)
        // 指定顶点数组
        GLES20.glVertexAttribPointer(
            texture2DProgram.aPositionLoc,
            2,
            GLES20.GL_FLOAT,
            false,
            0,
            shapeVerticesBuffer
        )
        GLES20.glVertexAttribPointer(
            texture2DProgram.aTextureCoordLoc,
            2,
            GLES20.GL_FLOAT,
            false,
            0,
            textureVerticesBuffer
        )
        // 清除颜色缓冲区
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f)
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT)
        // 从数组中获得数据渲染图元
        GLES20.glDrawElements(
            GLES20.GL_TRIANGLES,
            drawIndicesBuffer.limit(),
            GLES20.GL_UNSIGNED_SHORT,
            drawIndicesBuffer
        )

        GLES20.glFinish()
        // 解绑一系列
        GLES20.glDisableVertexAttribArray(texture2DProgram.aPositionLoc)
        GLES20.glDisableVertexAttribArray(texture2DProgram.aTextureCoordLoc)
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0)
        GLES20.glUseProgram(0)
    }

    /**
     * bitmap转2D纹理
     *
     * @param image
     * @param reUseTexture
     * @return 纹理id
     */
    fun loadTexture(image: Bitmap, reUseTexture: Int): Int {
        val textures = IntArray(1)
        if (reUseTexture == NO_TEXTURE) {
            GLES20.glGenTextures(1, textures, 0)
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textures[0])
            GLES20.glTexParameterf(
                GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_MIN_FILTER,
                GLES20.GL_LINEAR.toFloat()
            )
            GLES20.glTexParameterf(
                GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_MAG_FILTER,
                GLES20.GL_LINEAR.toFloat()
            )
            GLES20.glTexParameterf(
                GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_WRAP_S,
                GLES20.GL_CLAMP_TO_EDGE.toFloat()
            )
            GLES20.glTexParameterf(
                GLES20.GL_TEXTURE_2D,
                GLES20.GL_TEXTURE_WRAP_T,
                GLES20.GL_CLAMP_TO_EDGE.toFloat()
            )
            GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, image, 0)
        } else {
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, reUseTexture)
            GLUtils.texSubImage2D(GLES20.GL_TEXTURE_2D, 0, 0, 0, image)
            textures[0] = reUseTexture
        }
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0)
        return textures[0]
    }

}
```

**Texture2DProgram **2D纹理程序封装代码

```kotlin
package com.hezb.live.recorder.gles

import android.opengl.GLES20

/**
 * Project Name: AndroidScreenLive
 * File Name:    Texture2dProgram
 *
 * Description: 2D纹理程序.
 *
 * @author  hezhubo
 * @date    2022年07月19日 21:16
 */
class Texture2DProgram(vertexShaderCode: String = VERTEX_SHADER, fragmentShaderCode: String) {

    companion object {
        const val VERTEX_SHADER = "" +
                "attribute vec4 aPosition;\n" +
                "attribute vec2 aTextureCoord;\n" +
                "varying vec2 vTextureCoord;\n" +
                "void main(){\n" +
                "    gl_Position   = aPosition;\n" +
                "    vTextureCoord = aTextureCoord;\n" +
                "}"

        const val FRAGMENT_SHADER_2D = "" +
                "precision highp float;\n" +
                "varying highp vec2 vTextureCoord;\n" +
                "uniform sampler2D uTexture;\n" +
                "void main(){\n" +
                "    gl_FragColor = texture2D(uTexture, vTextureCoord);\n" +
                "}"
        const val FRAGMENT_SHADER_SOURCE2D = "" +
                "#extension GL_OES_EGL_image_external : require\n" +
                "precision highp float;\n" +
                "varying highp vec2 vTextureCoord;\n" +
                "uniform samplerExternalOES uTexture;\n" +
                "void main(){\n" +
                "    gl_FragColor = texture2D(uTexture, vTextureCoord);\n" +
                "}"
    }

    var program: Int = 0
        private set

    var aPositionLoc: Int = 0
        private set

    var aTextureCoordLoc: Int = 0
        private set

    var uTextureLoc: Int = 0
        private set

    init {
        program = GlUtil.createProgram(vertexShaderCode, fragmentShaderCode)
        if (program == 0) {
            throw RuntimeException("unable to create program!")
        }
        GLES20.glUseProgram(program)
        // 获取顶点属性变量
        aPositionLoc = GLES20.glGetAttribLocation(program, "aPosition")
        aTextureCoordLoc = GLES20.glGetAttribLocation(program, "aTextureCoord")
        // 获取统一变量的位置
        uTextureLoc = GLES20.glGetUniformLocation(program, "uTexture")
    }

}
```

### 音视频编解码输出

**MediaCodec.dequeueOutputBuffer(bufferInfo, timeoutUs)** 通过调用此方法等待编解码器对输入数据进行编解码后，编解码信息输出至bufferInfo，此方法返回Int值(大于等于0时)为内部OutputBuffer数组下标，通过**MediaCodec.getOutputBuffer(index)**获取到编解码后的数据

```kotlin
// 编码器输出示例
val bufferInfo = MediaCodec.BufferInfo()
val eobIndex = encoderCodec.dequeueOutputBuffer(bufferInfo, 5000) // 阻塞线程，超时时间为5秒
when (eobIndex) {
    MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED -> {
      	// 输出缓冲区已更改（已废弃）
    }
    MediaCodec.INFO_TRY_AGAIN_LATER -> {
      	// 编解码超时
    }
    MediaCodec.INFO_OUTPUT_FORMAT_CHANGED -> {
      	// 输出格式变更，一般来说编解码器启动后会首先返回输出格式
      	// 如果使用MediaMuxer进行混合封装输出，此处应该把输出格式添加到混合器轨道MediaMuxer.addTrack(encoderCodec.outputFormat)
    }
    else -> {
        if (outputBufferInfo.flags != MediaCodec.BUFFER_FLAG_CODEC_CONFIG && outputBufferInfo.size != 0) {
            try {
              val encodedData = encoder.getOutputBuffer(eobIndex) ?: return@let
              encodedData.position(outputBufferInfo.offset + 4) // H264 NALU: 00 00 00 01(4字节) 起始码(4字节)；后一个字节为NALU type
              encodedData.limit(outputBufferInfo.offset + outputBufferInfo.size)
              // 输出编码数据
            } catch (e : Exception) {
            }
        }
        encoder.releaseOutputBuffer(eobIndex, false)

        if (outputBufferInfo.flags and MediaCodec.BUFFER_FLAG_END_OF_STREAM != 0) {
          isRunning = false
        }
    }
}
```



