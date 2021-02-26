# android webrtc 音视频会议海思荣耀硬编解决方案

 公司测试手机荣耀手机，本地摄像头能正常显示，但是过一段时间，对方却看到了本地的数据显示某一帧停止的画面，此时查看本地摄像头图像，发现还是正常的。

  通过分析日志，可以知道，摄像头编码的时候报错了，查看日志，提示：

```log
Dropped frame, encoder queue full
```

  分析webrtc源码，如下

```java
if (outputBuilders.size() > MAX_ENCODER_Q_SIZE) {
  // Too many frames in the encoder.  Drop this frame.
  Logging.e(TAG, "Dropped frame, encoder queue full");
  return VideoCodecStatus.NO_OUTPUT; // See webrtc bug 2887.
}
```

  通过查看webrtc bug 2887说明如下：

When a VideoDecoder::Decode() or VideoEncoder::Encode() implementation falls behind (e.g. b/c a HW codec ran out of input slots) it's unclear what should be done.  Today seemingly the right answer is to drop the frame on the floor and return OK from the respective functions above and hope for future recovery.  It would be better if there was a prescribed error code for {En,De}code() to return to indicate to the caller that the frame was dropped and that future load should be reduced (at least temporarily).

(see google-internal thread at [go/rkvrz](http://go/rkvrz) for a bit more color)

  即丢帧或者返回ok并重启恢复

  查看上面出错处理有如下代码如下：

```java
VideoCodecStatus status = resetCodec(frameWidth, frameHeight, shouldUseSurfaceMode);
if (status != VideoCodecStatus.OK) {
  return status;
}
```

  即重启一下编码器，但是这是安卓通用视频编码代码，不能全部都这么处理，要做下处理，通话打印测试，发现编码器名称为"OMX.hisi.video.encoder.avc"，即我们发现报错并且编码器是海思的即执行reset操作，

  最后代码如下：

```java
if (outputBuilders.size() > MAX_ENCODER_Q_SIZE) {
  // Too many frames in the encoder.  Drop this frame.
  Logging.e(TAG, "Dropped frame, encoder queue full");
  if (bHisi) {
    VideoCodecStatus status = resetCodec(frameWidth, frameHeight, shouldUseSurfaceMode);
    if (status != VideoCodecStatus.OK) {
      return status;
    }
  }
  return VideoCodecStatus.NO_OUTPUT; // See webrtc bug 2887.
}
```

