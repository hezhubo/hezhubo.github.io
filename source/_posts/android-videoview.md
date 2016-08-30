---
title: android多媒体播放器之VideoView
date: 2016-08-28 12:26:39
tags: [videoview,多媒体,音视频]
---
&nbsp;&nbsp;&nbsp;&nbsp;在需要播放视频的时候，我们会往往会直接使用VideoView，因为方便，简单几行代码就搞定了。
```java
VideoView videoView; // 通过xml资源或是使用代码自行创建
MediaController mediaController = new MediaController(this); // 封装了播放控制的一个布局（包含播放/暂停，时长显示，进度条等），this 上下文
videoView.setMediaController(mediaController); // 设置控制器
videoView.setVideoPath("/mnt/sdcard/test.mp4"); // 视频路径，支持本地文件，http，rtsp协议
videoView.start(); // 开始播放
```
</br>
**VideoView其实是SurfaceView+MediaPlayer的封装，下面来分析一下源码的实现。**
### *基于API 23的源码分析：*
#### 结构关系
```java
public class VideoView extends SurfaceView implements MediaPlayerControl, SubtitleController.Anchor {
}
```
我们可以看到VideoView继承SurfaceView，实现了两接口：MediaPlayerControl和SubtitleController.Anchor。MediaPlayerControl就是android.widget.MediaController的内部接口，MediaController就是通过这个接口来控制播放器的。至于SubtitleController.Anchor这个接口是和字幕有关的，具体怎么用我暂时不清楚。
下面看一下MediaPlayerControl的源码：
```java
 public interface MediaPlayerControl {
    void    start(); // 播放
    void    pause(); // 暂停
    int     getDuration(); // 获取总时长
    int     getCurrentPosition(); // 获取当前播放位置
    void    seekTo(int pos); // 跳转到指定位置
    boolean isPlaying(); // 是否在播放
    int     getBufferPercentage(); // 获取总缓冲百分比
    boolean canPause(); // 能否暂停
    boolean canSeekBackward(); // 能否向后跳转
    boolean canSeekForward(); // 能否向前跳转

    /**
     * Get the audio session id for the player used by this VideoView. This can be used to
     * apply audio effects to the audio track of a video.
     * @return The audio session, or 0 if there was an error.
     */
    int     getAudioSessionId();
}
```

#### 成员变量
```java
// settable by the client
private Uri         mUri; // 视频的Uri
private Map<String, String> mHeaders; // 请求头

// 所有有可能的播放状态
private static final int STATE_ERROR              = -1; // 错误
private static final int STATE_IDLE               = 0; // 初始
private static final int STATE_PREPARING          = 1; // 预处理中
private static final int STATE_PREPARED           = 2; // 预处理完成
private static final int STATE_PLAYING            = 3; // 播放中
private static final int STATE_PAUSED             = 4; // 暂停
private static final int STATE_PLAYBACK_COMPLETED = 5; // 播放完成

private int mCurrentState = STATE_IDLE; // 当前状态
private int mTargetState  = STATE_IDLE; // 目标状态

// All the stuff we need for playing and showing a video
private SurfaceHolder mSurfaceHolder = null; // Surface持有者
private MediaPlayer mMediaPlayer = null; // 真正的播放器
private int         mAudioSession; // 音频 Session id
private int         mVideoWidth; // 视频宽
private int         mVideoHeight; // 视频高
private int         mSurfaceWidth; // 显示(绘制)宽
private int         mSurfaceHeight; // 显示(绘制)高
private MediaController mMediaController; // 控制层控制器
private OnCompletionListener mOnCompletionListener; // 播放完成监听器
private MediaPlayer.OnPreparedListener mOnPreparedListener; // 预处理监听器
private int         mCurrentBufferPercentage; // 当前缓冲百分比 
private OnErrorListener mOnErrorListener; // 错误监听器
private OnInfoListener  mOnInfoListener; // 播放(过程)信息监听器
private int         mSeekWhenPrepared;  // 预处理完成跳转位置
private boolean     mCanPause; // 能否暂停
private boolean     mCanSeekBack; // 能否向后跳转
private boolean     mCanSeekForward; // 能否向前跳转

/** Subtitle rendering widget overlaid on top of the video. */
private RenderingWidget mSubtitleWidget; // 字幕渲染组件

/** Listener for changes to subtitle data, used to redraw when needed. */
private RenderingWidget.OnChangedListener mSubtitlesChangedListener;
```

#### 初始化方法
```java
private void initVideoView() {
    mVideoWidth = 0;
    mVideoHeight = 0;
    getHolder().addCallback(mSHCallback); // 给Holder添加回调
    getHolder().setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS); // 这个可以不设置
    setFocusable(true); // 设置可以获取焦点
    setFocusableInTouchMode(true); // 设置触摸可以获取焦点
    requestFocus(); // 请求焦点
    mPendingSubtitleTracks = new Vector<Pair<InputStream, MediaFormat>>(); // 字幕相关 字幕轨道表
    // 当前状态与目标状态均为初始状态
    mCurrentState = STATE_IDLE;
    mTargetState  = STATE_IDLE;
}
```
其中mSHCallback代码如下：
```java
SurfaceHolder.Callback mSHCallback = new SurfaceHolder.Callback()
{
    public void surfaceChanged(SurfaceHolder holder, int format,
                                int w, int h)
    { // 在surface发生改变时回调此方法
        mSurfaceWidth = w; // 设置surface宽
        mSurfaceHeight = h; // 设置surface高
        boolean isValidState =  (mTargetState == STATE_PLAYING); // 若目标状态是播放状态，则记为有效状态
        boolean hasValidSize = (mVideoWidth == w && mVideoHeight == h); // 若视频宽高与surface宽高一致，则记为有效尺寸
        if (mMediaPlayer != null && isValidState && hasValidSize) { // 播放器可用，并且状态与尺寸均有效，则开始播放
            if (mSeekWhenPrepared != 0) { // 这是播放前检查是否要跳转
                seekTo(mSeekWhenPrepared); // 跳转到指定位置
            }
            start(); // 开始播放
        }
    }

    public void surfaceCreated(SurfaceHolder holder)
    { // 在surface创建时回调此方法
        mSurfaceHolder = holder;
        openVideo(); // 打开播放器
    }

    public void surfaceDestroyed(SurfaceHolder holder)
    { // 在surface销毁时回调此方法，一旦销毁就不能再使用
        mSurfaceHolder = null;
        if (mMediaController != null) mMediaController.hide(); // 隐藏控制层
        release(true); // 释放资源
    }
};
```
#### SurfaceView绘制部分
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) { // 测量宽高
    int width = getDefaultSize(mVideoWidth, widthMeasureSpec);
    int height = getDefaultSize(mVideoHeight, heightMeasureSpec);
    if (mVideoWidth > 0 && mVideoHeight > 0) {

        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

        if (widthSpecMode == MeasureSpec.EXACTLY && heightSpecMode == MeasureSpec.EXACTLY) {
            // the size is fixed
            width = widthSpecSize;
            height = heightSpecSize;

            // for compatibility, we adjust size based on aspect ratio
            if ( mVideoWidth * height  < width * mVideoHeight ) {
                //Log.i("@@@", "image too wide, correcting");
                width = height * mVideoWidth / mVideoHeight;
            } else if ( mVideoWidth * height  > width * mVideoHeight ) {
                //Log.i("@@@", "image too tall, correcting");
                height = width * mVideoHeight / mVideoWidth;
            }
        } else if (widthSpecMode == MeasureSpec.EXACTLY) {
            // only the width is fixed, adjust the height to match aspect ratio if possible
            width = widthSpecSize;
            height = width * mVideoHeight / mVideoWidth;
            if (heightSpecMode == MeasureSpec.AT_MOST && height > heightSpecSize) {
                // couldn't match aspect ratio within the constraints
                height = heightSpecSize;
            }
        } else if (heightSpecMode == MeasureSpec.EXACTLY) {
            // only the height is fixed, adjust the width to match aspect ratio if possible
            height = heightSpecSize;
            width = height * mVideoWidth / mVideoHeight;
            if (widthSpecMode == MeasureSpec.AT_MOST && width > widthSpecSize) {
                // couldn't match aspect ratio within the constraints
                width = widthSpecSize;
            }
        } else {
            // neither the width nor the height are fixed, try to use actual video size
            width = mVideoWidth;
            height = mVideoHeight;
            if (heightSpecMode == MeasureSpec.AT_MOST && height > heightSpecSize) {
                // too tall, decrease both width and height
                height = heightSpecSize;
                width = height * mVideoWidth / mVideoHeight;
            }
            if (widthSpecMode == MeasureSpec.AT_MOST && width > widthSpecSize) {
                // too wide, decrease both width and height
                width = widthSpecSize;
                height = width * mVideoHeight / mVideoWidth;
            }
        }
    } else {
        // no size yet, just adopt the given spec sizes
    }
    setMeasuredDimension(width, height);
}
```