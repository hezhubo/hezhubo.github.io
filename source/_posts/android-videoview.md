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
### *基于API 23的主要源码分析：*
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
#### SurfaceView测量部分
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) { // 测量宽高
    // 根据视频尺寸和布局宽高计算宽高
    int width = getDefaultSize(mVideoWidth, widthMeasureSpec);
    int height = getDefaultSize(mVideoHeight, heightMeasureSpec);
    if (mVideoWidth > 0 && mVideoHeight > 0) { // 当视频宽高可用时
        // 计算布局设定的大小
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

        // 下列条件父布局宽高是否限定子布局宽高
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
    setMeasuredDimension(width, height); // 存储测量的宽高
}
```
#### 内部监听器
##### 视频尺寸变化监听器
```java
MediaPlayer.OnVideoSizeChangedListener mSizeChangedListener =
    new MediaPlayer.OnVideoSizeChangedListener() {
        public void onVideoSizeChanged(MediaPlayer mp, int width, int height) {
            mVideoWidth = mp.getVideoWidth();
            mVideoHeight = mp.getVideoHeight();
            if (mVideoWidth != 0 && mVideoHeight != 0) {
                // 设置surface固定尺寸
                getHolder().setFixedSize(mVideoWidth, mVideoHeight);
                requestLayout(); // 请求重新测量布局
            }
        }
};
```
##### 视频预处理监听器
```java
MediaPlayer.OnPreparedListener mPreparedListener = new MediaPlayer.OnPreparedListener() {
    public void onPrepared(MediaPlayer mp) {
        mCurrentState = STATE_PREPARED; // 当前状态已预处理完成

        // Get the capabilities of the player for this stream
        Metadata data = mp.getMetadata(MediaPlayer.METADATA_ALL,
                                  MediaPlayer.BYPASS_METADATA_FILTER);

        if (data != null) { // 解析获取的视频信息
            mCanPause = !data.has(Metadata.PAUSE_AVAILABLE)
                    || data.getBoolean(Metadata.PAUSE_AVAILABLE);
            mCanSeekBack = !data.has(Metadata.SEEK_BACKWARD_AVAILABLE)
                    || data.getBoolean(Metadata.SEEK_BACKWARD_AVAILABLE);
            mCanSeekForward = !data.has(Metadata.SEEK_FORWARD_AVAILABLE)
                    || data.getBoolean(Metadata.SEEK_FORWARD_AVAILABLE);
        } else { // 视频信息获取失败，下面操作默认都可以
            mCanPause = mCanSeekBack = mCanSeekForward = true;
        }

        if (mOnPreparedListener != null) {
            mOnPreparedListener.onPrepared(mMediaPlayer); // 告诉外部的监听器
        }
        if (mMediaController != null) {
            mMediaController.setEnabled(true); // 当前控制器可用
        }
        mVideoWidth = mp.getVideoWidth();
        mVideoHeight = mp.getVideoHeight();

        int seekToPosition = mSeekWhenPrepared;  // mSeekWhenPrepared may be changed after seekTo() call
        if (seekToPosition != 0) {
            seekTo(seekToPosition); // 跳转到指定位置
        }
        if (mVideoWidth != 0 && mVideoHeight != 0) {
            getHolder().setFixedSize(mVideoWidth, mVideoHeight); // 设置surface固定尺寸
            if (mSurfaceWidth == mVideoWidth && mSurfaceHeight == mVideoHeight) {
                // We didn't actually change the size (it was already at the size
                // we need), so we won't get a "surface changed" callback, so
                // start the video here instead of in the callback.
                if (mTargetState == STATE_PLAYING) { // 当目标状态为播放状态时
                    start(); // 开始播放
                    if (mMediaController != null) {
                        mMediaController.show(); // 展示控制器UI
                    }
                } else if (!isPlaying() &&
                           (seekToPosition != 0 || getCurrentPosition() > 0)) {
                   // 不是在播放状态并且当前进度不为0时，展示控制器UI
                   if (mMediaController != null) {
                       // Show the media controls when we're paused into a video and make 'em stick.
                       mMediaController.show(0);
                   }
               }
            }
        } else {
            // We don't know the video size yet, but should start anyway.
            // The video size might be reported to us later.
            if (mTargetState == STATE_PLAYING) { // 当目标状态为播放状态时
                start(); // 开始播放
            }
        }
    }
};
```
##### 视频播放完成监听器
```java
private MediaPlayer.OnCompletionListener mCompletionListener =
    new MediaPlayer.OnCompletionListener() {
    public void onCompletion(MediaPlayer mp) {
        // 当前状态与目标状态为播放完成状态
        mCurrentState = STATE_PLAYBACK_COMPLETED;
        mTargetState = STATE_PLAYBACK_COMPLETED;
        if (mMediaController != null) {
            mMediaController.hide(); // 隐藏控制层
        }
        if (mOnCompletionListener != null) {
            mOnCompletionListener.onCompletion(mMediaPlayer); // 通知外部的监听器
        }
    }
};
```
##### 视频播放(过程)信息监听器
```java
private MediaPlayer.OnInfoListener mInfoListener =
    new MediaPlayer.OnInfoListener() {
    public  boolean onInfo(MediaPlayer mp, int arg1, int arg2) {
        // 这里没做任何处理，直接通知外部的监听器
        if (mOnInfoListener != null) {
            mOnInfoListener.onInfo(mp, arg1, arg2);
        }
        return true;
    }
};
```
##### 视频播放错误监听器
```java
private MediaPlayer.OnErrorListener mErrorListener =
    new MediaPlayer.OnErrorListener() {
    public boolean onError(MediaPlayer mp, int framework_err, int impl_err) {
        Log.d(TAG, "Error: " + framework_err + "," + impl_err);
        // 当前状态与目标状态为错误状态
        mCurrentState = STATE_ERROR;
        mTargetState = STATE_ERROR;
        if (mMediaController != null) {
            mMediaController.hide(); // 隐藏控制层UI
        }

        /* If an error handler has been supplied, use it and finish. */
        if (mOnErrorListener != null) {
            if (mOnErrorListener.onError(mMediaPlayer, framework_err, impl_err)) {
                // 通知外部的监听器，又外部来处理错误，不往下执行
                return true;
            }
        }

        /* Otherwise, pop up an error dialog so the user knows that
         * something bad has happened. Only try and pop up the dialog
         * if we're attached to a window. When we're going away and no
         * longer have a window, don't bother showing the user an error.
         */
        if (getWindowToken() != null) {
            Resources r = mContext.getResources();
            int messageId;

            if (framework_err == MediaPlayer.MEDIA_ERROR_NOT_VALID_FOR_PROGRESSIVE_PLAYBACK) {
                messageId = com.android.internal.R.string.VideoView_error_text_invalid_progressive_playback;
            } else {
                messageId = com.android.internal.R.string.VideoView_error_text_unknown;
            }

            new AlertDialog.Builder(mContext)
                    .setMessage(messageId)
                    .setPositiveButton(com.android.internal.R.string.VideoView_error_button,
                            new DialogInterface.OnClickListener() {
                                public void onClick(DialogInterface dialog, int whichButton) {
                                    /* If we get here, there is no onError listener, so
                                     * at least inform them that the video is over.
                                     */
                                    if (mOnCompletionListener != null) {
                                        mOnCompletionListener.onCompletion(mMediaPlayer);
                                    }
                                }
                            })
                    .setCancelable(false)
                    .show();
        }
        return true;
    }
};
```
##### 视频缓冲监听器
```java
private MediaPlayer.OnBufferingUpdateListener mBufferingUpdateListener =
    new MediaPlayer.OnBufferingUpdateListener() {
    public void onBufferingUpdate(MediaPlayer mp, int percent) {
        mCurrentBufferPercentage = percent; // 记录当前缓冲百分比
    }
};
```
#### 触碰及键盘事件
```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    // 触摸事件
    if (isInPlaybackState() && mMediaController != null) { // 当前状态为可用状态并且控制器存在
        toggleMediaControlsVisiblity(); // 显示/隐藏控制层UI
    }
    return false;
}

@Override
public boolean onTrackballEvent(MotionEvent ev) {
    // 轨迹球事件
    if (isInPlaybackState() && mMediaController != null) { // 当前状态为可用状态并且控制器存在
        toggleMediaControlsVisiblity(); // 显示/隐藏控制层UI
    }
    return false;
}

/**
 * 判断是否为可用状态
 * 播放器不为空
 * 当前状态不为错误，初始，预处理中
 */
private boolean isInPlaybackState() {
    return (mMediaPlayer != null &&
            mCurrentState != STATE_ERROR &&
            mCurrentState != STATE_IDLE &&
            mCurrentState != STATE_PREPARING);
}

@Override
public boolean onKeyDown(int keyCode, KeyEvent event)
{
    // 主要是用于显示/隐藏控制器UI,以及处理播放暂停
    // 判断按键是否为需要处理的按键
    boolean isKeyCodeSupported = keyCode != KeyEvent.KEYCODE_BACK &&
                                 keyCode != KeyEvent.KEYCODE_VOLUME_UP &&
                                 keyCode != KeyEvent.KEYCODE_VOLUME_DOWN &&
                                 keyCode != KeyEvent.KEYCODE_VOLUME_MUTE &&
                                 keyCode != KeyEvent.KEYCODE_MENU &&
                                 keyCode != KeyEvent.KEYCODE_CALL &&
                                 keyCode != KeyEvent.KEYCODE_ENDCALL;
    if (isInPlaybackState() && isKeyCodeSupported && mMediaController != null) {
        if (keyCode == KeyEvent.KEYCODE_HEADSETHOOK ||
                keyCode == KeyEvent.KEYCODE_MEDIA_PLAY_PAUSE) {
            if (mMediaPlayer.isPlaying()) {
                pause();
                mMediaController.show();
            } else {
                start();
                mMediaController.hide();
            }
            return true;
        } else if (keyCode == KeyEvent.KEYCODE_MEDIA_PLAY) {
            if (!mMediaPlayer.isPlaying()) {
                start();
                mMediaController.hide();
            }
            return true;
        } else if (keyCode == KeyEvent.KEYCODE_MEDIA_STOP
                || keyCode == KeyEvent.KEYCODE_MEDIA_PAUSE) {
            if (mMediaPlayer.isPlaying()) {
                pause();
                mMediaController.show();
            }
            return true;
        } else {
            toggleMediaControlsVisiblity();
        }
    }

    return super.onKeyDown(keyCode, event);
}

/**
 * 控制器UI显示与隐藏相互切换
 */
private void toggleMediaControlsVisiblity() {
    if (mMediaController.isShowing()) {
        mMediaController.hide();
    } else {
        mMediaController.show();
    }
}
```
#### 主要操作方法
##### 设置视频源路径
```java
/**
 * Sets video path.
 *
 * @param path the path of the video.
 */
public void setVideoPath(String path) {
    setVideoURI(Uri.parse(path));
}

/**
 * Sets video URI.
 *
 * @param uri the URI of the video.
 */
public void setVideoURI(Uri uri) {
    setVideoURI(uri, null);
}

/**
 * Sets video URI using specific headers.
 *
 * @param uri     the URI of the video.
 * @param headers the headers for the URI request.
 *                Note that the cross domain redirection is allowed by default, but that can be
 *                changed with key/value pairs through the headers parameter with
 *                "android-allow-cross-domain-redirect" as the key and "0" or "1" as the value
 *                to disallow or allow cross domain redirection.
 */
public void setVideoURI(Uri uri, Map<String, String> headers) {
    mUri = uri; // 设置视频的Uri
    mHeaders = headers; // 设置请求头
    mSeekWhenPrepared = 0; // 设置预处理完成不跳转
    openVideo(); // 打开播放器并设置相应参数
    requestLayout(); // 请求重新测量布局
    invalidate(); // 请求重绘
}

/**
 * 最重要的方法————开启视频
 * 初始化MediaPlayer及其参数
 */
private void openVideo() {
    if (mUri == null || mSurfaceHolder == null) {
        // not ready for playback just yet, will try again later
        return;
    }
    // we shouldn't clear the target state, because somebody might have
    // called start() previously
    release(false); // 不改变状态地释放资源

    // 获取音频管理器并请求获得音频焦点，销毁时需要释放焦点
    AudioManager am = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
    am.requestAudioFocus(null, AudioManager.STREAM_MUSIC, AudioManager.AUDIOFOCUS_GAIN);

    try {
        mMediaPlayer = new MediaPlayer();
        // 以下设置字幕相关的
        // TODO: create SubtitleController in MediaPlayer, but we need
        // a context for the subtitle renderers
        final Context context = getContext();
        final SubtitleController controller = new SubtitleController(
                context, mMediaPlayer.getMediaTimeProvider(), mMediaPlayer);
        controller.registerRenderer(new WebVttRenderer(context));
        controller.registerRenderer(new TtmlRenderer(context));
        controller.registerRenderer(new ClosedCaptionRenderer(context));
        mMediaPlayer.setSubtitleAnchor(controller, this);

        // 设置音频的SessionId
        if (mAudioSession != 0) {
            mMediaPlayer.setAudioSessionId(mAudioSession);
        } else {
            mAudioSession = mMediaPlayer.getAudioSessionId();
        }

        // 设置各监听器
        mMediaPlayer.setOnPreparedListener(mPreparedListener);
        mMediaPlayer.setOnVideoSizeChangedListener(mSizeChangedListener);
        mMediaPlayer.setOnCompletionListener(mCompletionListener);
        mMediaPlayer.setOnErrorListener(mErrorListener);
        mMediaPlayer.setOnInfoListener(mInfoListener);
        mMediaPlayer.setOnBufferingUpdateListener(mBufferingUpdateListener);

        mCurrentBufferPercentage = 0; // 缓冲进度为0
        mMediaPlayer.setDataSource(mContext, mUri, mHeaders); // 设置播放地址等参数
        mMediaPlayer.setDisplay(mSurfaceHolder); // 设置Surface持有者，用于绘制视频
        mMediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC); // 设置音频流类型
        mMediaPlayer.setScreenOnWhilePlaying(true); // 设置当播放保持屏幕高亮
        mMediaPlayer.prepareAsync(); // 开始异步预处理

        // 添加字幕流
        for (Pair<InputStream, MediaFormat> pending: mPendingSubtitleTracks) {
            try {
                mMediaPlayer.addSubtitleSource(pending.first, pending.second);
            } catch (IllegalStateException e) {
                mInfoListener.onInfo(
                        mMediaPlayer, MediaPlayer.MEDIA_INFO_UNSUPPORTED_SUBTITLE, 0);
            }
        }

        // we don't set the target state here either, but preserve the
        // target state that was there before.
        mCurrentState = STATE_PREPARING; // 当前为预处理中状态
        attachMediaController(); // 添加控制器（若没有设置，则不添加）
    } catch (IOException ex) {
        Log.w(TAG, "Unable to open content: " + mUri, ex);
        mCurrentState = STATE_ERROR;
        mTargetState = STATE_ERROR;
        mErrorListener.onError(mMediaPlayer, MediaPlayer.MEDIA_ERROR_UNKNOWN, 0);
        return;
    } catch (IllegalArgumentException ex) {
        Log.w(TAG, "Unable to open content: " + mUri, ex);
        mCurrentState = STATE_ERROR;
        mTargetState = STATE_ERROR;
        mErrorListener.onError(mMediaPlayer, MediaPlayer.MEDIA_ERROR_UNKNOWN, 0);
        return;
    } finally {
        mPendingSubtitleTracks.clear();
    }
}

/*
 * release the media player in any state
 */
private void release(boolean cleartargetstate) {
    if (mMediaPlayer != null) {
        mMediaPlayer.reset(); // 重置播放器
        mMediaPlayer.release(); // 是否播放器
        mMediaPlayer = null;
        mPendingSubtitleTracks.clear(); // 释放字幕
        mCurrentState = STATE_IDLE; // 当前状态为初始状态
        if (cleartargetstate) { // 需要清楚目标状态
            mTargetState  = STATE_IDLE; // 目标状态为初始状态
        }
        AudioManager am = (AudioManager) mContext.getSystemService(Context.AUDIO_SERVICE);
        am.abandonAudioFocus(null); // 释放音频焦点
    }
}
```
##### 播放
```java
@Override
public void start() {
    if (isInPlaybackState()) { // 播放器可以时
        mMediaPlayer.start(); // 开始播放
        mCurrentState = STATE_PLAYING; // 当前状态为播放状态
    }
    mTargetState = STATE_PLAYING; // 目标状态为播放状态
}
```
##### 暂停
```java
@Override
public void pause() {
    if (isInPlaybackState()) { // 播放器可以时
        if (mMediaPlayer.isPlaying()) { // 正在播放
            mMediaPlayer.pause(); // 暂停
            mCurrentState = STATE_PAUSED; // 当前状态为暂停状态
        }
    }
    mTargetState = STATE_PAUSED; // 目标状态为暂停状态
}
```
##### 挂起播放器
```java
// 此方法一般在Activity进入onStop()状态时调用，不改变状态地释放资源
public void suspend() {
    release(false);
}
```
##### 恢复播放器
```java
// 此方法一般在Activity从后台调到前台进入onResume()状态时调用
public void resume() {
    openVideo();
}
```
##### 其他方法
```java
int getDuration() // 获取时长
int getCurrentPosition() // 获取当前播放进度
void seekTo(int msec) // 跳转到指定位置
boolean isPlaying() // 视频是否在播放
getBufferPercentage() // 获取缓冲百分比
```
其他 控制器UI加入 及 字幕相关 等方法就不一一分析了。可以自行查看[VideoView源码](https://developer.android.com/reference/android/widget/VideoView.html)