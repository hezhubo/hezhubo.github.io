---
title: FFmpeg 常用命令
date: 2016-07-05 09:35:34
tags: [音视频,FFmpeg]
---
### 常用命令
#### 分离视频音频流
```bash
ffmpeg -i input_file -vcodec copy -an output_file_video　　// 分离视频流
ffmpeg -i input_file -acodec copy -vn output_file_audio　　// 分离音频流
```
</br>
#### 视频裁剪
```bash
ffmpeg -ss 0:1:30 -t 0:0:20 -i input.mp4 -vcodec copy -acodec copy output.mp4    // 时间格式可以是 00:00:00.000 或 x.xxx(单位秒)
```
</br>
#### 视频合成
```bash
ffmpeg -i "1.mp4" -vcodec copy -acodec copy -vbsf h264_mp4toannexb "1.ts"
ffmpeg -i "2.mp4" -vcodec copy -acodec copy -vbsf h264_mp4toannexb "2.ts"
ffmpeg -i "concat:1.ts|2.ts" -acodec copy -vcodec copy -absf aac_adtstoasc "output.mp4"    // 若只有1个ts,不需要concat:
```
</br>
#### 视频截图
```bash
ffmpeg -i "input.mp4" -r 2 -start_number 1 -vframes 10 -s 60x60 "thumb_%d.jpg"    // 连续截取图片，设置帧率为2fps，截取10张，也就是截取了5秒的视频截图，截图保存的名称的第一张为thumb_1.jpg，往后自动累加
ffmpeg -ss 00:56 -i "input.mp4" -f image2 -s 100x100 "thumb.jpg"    // 截取指定时间（00:56）的画面
```
</br>
#### 视频录制
```bash
ffmpeg -i "rtmp://192.168.1.100:1935/live/demo" –vcodec copy "output.mp4"
```
</br>
#### YUV序列播放
```bash
ffplay -f rawvideo -video_size 1920x1080 input.yuv
```
</br>
#### YUV序列转AVI
```bash
ffmpeg –s w*h –pix_fmt yuv420p –i input.yuv –vcodec mpeg4 output.avi    // w/h 视频宽/高
```
</br>
#### 加水印
```bash
ffmpeg -i input.mp4 -i logo.png -filter_complex "overlay=10:5" out.mp4    // 左上角 距离 左边距 10  上边距 5
```
overlay参数:
左上角为 overlay=10:5  // 左边 10  上边 5
右上角为 overlay=main_w-overlay_w-10:5   // 右边 10  上边 5
右下角为 overlay=main_w-overlay_w-10:main_h-overlay_h-5    // 右边 10  下边 5
左下角为 overlay=10: main_h-overlay_h-5  // 左边 10  上边 5


----


### 常用参数说明
#### 主要参数：
**-i** 设定输入流

**-f** 设定输出格式

**-ss** 开始时间

**-t** 持续时间

**-y**  覆盖输出文件

#### 视频参数：
**-aspect** 设定画面的比例

**-b** 设定视频流量，默认为200Kbit/s

**-bitexact** 使用标准比特率

**-r** 设定帧速率，默认为25

**-s** 设定画面的宽与高

**-title** 设置标题

**-vn** 不处理视频

**-vcodec** 设定视频编解码器，未设定时则使用与输入流相同的编解码器

**-vframes** 设置转换多少帧(frame)的视频

**-qscale**  视频质量，取值0.01-255，约小质量越好 

**-qmin** 设定最小质量，与-qmax（设定最大质量）共用，比如-qmin 10 -qmax 31 

**-sameq** 使用和源同样的质量

#### 音频参数：
**-ar** 设定采样率

**-ac** 设定声音的Channel数

**-acodec** 设定声音编解码器，未设定时则使用与输入流相同的编解码器

**-ab** 设定声音比特率

**-an** 不处理音频

**-vol** 设定音量<百分比>，200即原来的2倍