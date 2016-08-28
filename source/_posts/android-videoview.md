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

// all possible internal states
private static final int STATE_ERROR              = -1;
private static final int STATE_IDLE               = 0;
private static final int STATE_PREPARING          = 1;
private static final int STATE_PREPARED           = 2;
private static final int STATE_PLAYING            = 3;
private static final int STATE_PAUSED             = 4;
private static final int STATE_PLAYBACK_COMPLETED = 5;

// mCurrentState is a VideoView object's current state.
// mTargetState is the state that a method caller intends to reach.
// For instance, regardless the VideoView object's current state,
// calling pause() intends to bring the object to a target state
// of STATE_PAUSED.
private int mCurrentState = STATE_IDLE;
private int mTargetState  = STATE_IDLE;

// All the stuff we need for playing and showing a video
private SurfaceHolder mSurfaceHolder = null;
private MediaPlayer mMediaPlayer = null;
private int         mAudioSession;
private int         mVideoWidth;
private int         mVideoHeight;
private int         mSurfaceWidth;
private int         mSurfaceHeight;
private MediaController mMediaController;
private OnCompletionListener mOnCompletionListener;
private MediaPlayer.OnPreparedListener mOnPreparedListener;
private int         mCurrentBufferPercentage;
private OnErrorListener mOnErrorListener;
private OnInfoListener  mOnInfoListener;
private int         mSeekWhenPrepared;  // recording the seek position while preparing
private boolean     mCanPause;
private boolean     mCanSeekBack;
private boolean     mCanSeekForward;

/** Subtitle rendering widget overlaid on top of the video. */
private RenderingWidget mSubtitleWidget;

/** Listener for changes to subtitle data, used to redraw when needed. */
private RenderingWidget.OnChangedListener mSubtitlesChangedListener;
```

#### 初始化方法
```java
private void initVideoView() {
    mVideoWidth = 0;
    mVideoHeight = 0;
    getHolder().addCallback(mSHCallback);
    getHolder().setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
    setFocusable(true);
    setFocusableInTouchMode(true);
    requestFocus();
    mPendingSubtitleTracks = new Vector<Pair<InputStream, MediaFormat>>();
    mCurrentState = STATE_IDLE;
    mTargetState  = STATE_IDLE;
}
```