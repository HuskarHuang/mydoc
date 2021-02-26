# android webrtc 红米手机(K30)视频编码失败解决方案

## 测试描述：手机兼容性(红米K30),群聊视频邀请使用红米K30的用户,接通通话后,K30用户头像显示为黑屏,需要开关摄像头

通过查看logcat日志，发现有如下日志：

2020-12-01 11:02:59.400 5492-5679/com.zx.zxlivsdemo I/ExtendedACodec: [OMX.qcom.video.encoder.vp8] configure, AMessage : AMessage(what = 'conf', target = 1) = {
int32_t color-format = 2130708361
int32_t i-frame-interval = 100
string mime = "video/x-vnd.on2.vp8"
int32_t width = 320
int32_t bitrate-mode = 2
int32_t bitrate = 172000
int32_t frame-rate = 60
int32_t height = 240
int32_t encoder = 1
}
2020-12-01 11:02:59.400 5492-5679/com.zx.zxlivsdemo W/OMXUtils: do not know color format 0x7f000789 = 2130708361
2020-12-01 11:02:59.406 5492-5646/com.zx.zxlivsdemo I/EglBase14Impl: Using OpenGL ES version 2
2020-12-01 11:02:59.418 5492-5646/com.zx.zxlivsdemo I/[video_encoder_wrapper.cc](http://video_encoder_wrapper.cc): (line 93): initEncode: 0
2020-12-01 11:02:59.418 5492-5646/com.zx.zxlivsdemo I/[quality_scaler.cc](http://quality_scaler.cc): (line 113): QP thresholds: low: 29, high: 95
2020-12-01 11:02:59.418 5492-5629/com.zx.zxlivsdemo I/[bitrate_allocator.cc](http://bitrate_allocator.cc): (line 519): UpdateAllocationLimits : total_requested_min_bitrate: 62 kbps, total_requested_padding_bitrate: 0 bps, total_requested_max_bitrate: 204 kbps
2020-12-01 11:02:59.422 5492-5679/com.zx.zxlivsdemo D/ACodec: dataspace changed to 0x10c40000 (R:2(Limited), P:3(BT601_6_625), M:3(BT601_6), T:3(SMPTE170M)) (R:2(Limited), S:2(BT601_625), T:3(SMPTE_170M))
2020-12-01 11:02:59.425 5492-5679/com.zx.zxlivsdemo E/ACodec: [OMX.qcom.video.encoder.vp8] ERROR(0x80001009)
2020-12-01 11:02:59.425 5492-5679/com.zx.zxlivsdemo E/ACodec: signalError(omxError 0x80001009, internalError -2147483648)
**2020-12-01 11:02:59.425 5492-5678/com.zx.zxlivsdemo E/MediaCodec: Codec reported err 0x80001009, actionCode 0, while in state 6**
**2020-12-01 11:02:59.426 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: deliverOutput failed**
**2020-12-01 11:02:59.426 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: java.lang.IllegalStateException**
**2020-12-01 11:02:59.426 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: java.lang.IllegalStateException**
**at android.media.MediaCodec.native_dequeueOutputBuffer(Native Method)**
**at android.media.MediaCodec.dequeueOutputBuffer(MediaCodec.java:2789)**
**at org.webrtc.MediaCodecWrapperFactoryImpl$MediaCodecWrapperImpl.dequeueOutputBuffer(MediaCodecWrapperFactoryImpl.java:73)**
**at org.webrtc.HardwareVideoEncoder.deliverEncodedImage(HardwareVideoEncoder.java:539)**
**at org.webrtc.HardwareVideoEncoder$1.run(HardwareVideoEncoder.java:527)**
2020-12-01 11:02:59.427 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: deliverOutput failed
2020-12-01 11:02:59.427 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: java.lang.IllegalStateException
2020-12-01 11:02:59.427 5492-5685/com.zx.zxlivsdemo E/HardwareVideoEncoder: java.lang.IllegalStateException
at android.media.MediaCodec.native_dequeueOutputBuffer(Native Method)

从以上日志可以看出，使用了android的mediaCodec编码，即硬编码，而且是使用vp8的编码格式，编码的mine格式为**"video/x-vnd.on2.vp8"**，并且错误发生在webrtc底层源码里面

抛异常的代码如下：

src/java/org/webrtc/HardwareVideoEncoder.java

 protected void deliverEncodedImage() {
  outputThreadChecker.checkIsOnValidThread();
  try {
   MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
   **int index = codec.dequeueOutputBuffer(info, DEQUEUE_OUTPUT_BUFFER_TIMEOUT_US);**
   if (index < 0) {
    if (index == MediaCodec.INFO_OUTPUT_BUFFERS_CHANGED) {
     outputBuffersBusyCount.waitForZero();
     outputBuffers = codec.getOutputBuffers();
    }
    return;
   }

src/java/org/webrtc/MediaCodecWrapperFactoryImpl.java

  @Override
  public int dequeueOutputBuffer(BufferInfo info, long timeoutUs) {
   **return mediaCodec.dequeueOutputBuffer(info, timeoutUs);**
  }

通过分析deliverEncodedImage()的调用，可知他是在一个线程的一个while循环里面不断的调用

 private Thread createOutputThread() {
  return new Thread() {
   @Override
   public void run() {
    while (running) {
     **deliverEncodedImage**();
    }
    releaseCodecOnOutputThread();
   }
  };
 }

分析可以得知，此循环的功能是不断的将摄像头数据做为输入源，将vp8的编码做为输出源输出到一个outputBuffers 里面，然后输出的时候报错了

编码器的初始化启动代码如下，以下为根据webrtc源码做了相应分析：

src/java/org/webrtc/HardwareVideoEncoder.java

 private VideoCodecStatus initEncodeInternal() {
  encodeThreadChecker.checkIsOnValidThread();



  lastKeyFrameNs = -1;

  try {
   codec = mediaCodecWrapperFactory.createByCodecName(codecName); **// codecName为上层api设置传下来的vp8，即找到对应的meidaCodec编码器**


  } catch (IOException | IllegalArgumentException e) {
   Logging.e(TAG, "Cannot create media encoder " + codecName);
   return VideoCodecStatus.FALLBACK_SOFTWARE; **// 如果找不到对应的硬件编码器，改为软件编码器**
  }

  final int colorFormat = useSurfaceMode ? surfaceColorFormat : yuvColorFormat; **//useSurfaceMode为true，即编码器的输入源为摄像头离屏渲染把图像渲染到纹理texture中，共享其texture**

  try {
   MediaFormat format = MediaFormat.createVideoFormat(codecType.mimeType(), width, height); **//要编码视频的宽高**
   format.setInteger(MediaFormat.KEY_BIT_RATE, adjustedBitrate);//设置比特率
   format.setInteger(KEY_BITRATE_MODE, VIDEO_ControlRateConstant);**//public static final int BITRATE_MODE_CBR = 2;表示编码器会根据图像内容的复杂度（实际上是帧间变化量的大小）来动态调整输出码率，图像复杂则码率高，图像简单则码率低；**   

   format.setInteger(MediaFormat.KEY_COLOR_FORMAT, colorFormat); 
   format.setInteger(MediaFormat.KEY_FRAME_RATE, bitrateAdjuster.getCodecConfigFramerate()); 
   format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, keyFrameIntervalSec);**//两个I帧之间的间隔**
   if (codecType == VideoCodecType.H264) { **//如果编码器类型为h264，需要做额外的配置，即设置h264的profile与level**
    String profileLevelId = params.get(VideoCodecInfo.H264_FMTP_PROFILE_LEVEL_ID);
    if (profileLevelId == null) {
     profileLevelId = VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1;
    }
    switch (profileLevelId) {
     case VideoCodecInfo.H264_CONSTRAINED_HIGH_3_1:
      format.setInteger("profile", VIDEO_AVC_PROFILE_HIGH);
      format.setInteger("level", VIDEO_AVC_LEVEL_3);
      break;
     case VideoCodecInfo.H264_CONSTRAINED_BASELINE_3_1:
      break;
     default:
      Logging.w(TAG, "Unknown profile level id: " + profileLevelId);
    }
   }
   Logging.d(TAG, "Format: " + format);
   codec.configure(
     format, null /* surface */, null /* crypto */, MediaCodec.CONFIGURE_FLAG_ENCODE); **//配置编码器**

   if (useSurfaceMode) {
    textureEglBase = EglBase.createEgl14(sharedContext, EglBase.CONFIG_RECORDABLE);**//sharedContext摄像头创建的eglContex**
    textureInputSurface = codec.createInputSurface();**//编码器创建的surface，即opengle中只要共享摄像头的eglContex与texture给mediaCodec，便可实现纹理共享，然后绘制到编码器surface，编码器surface作为编码输入源**
    textureEglBase.createSurface(textureInputSurface);
    textureEglBase.makeCurrent();
   }

   codec.start();**//启动编码器**
   outputBuffers = codec.getOutputBuffers();**//拿到编码器编码后vp8的输出缓冲区**
  } catch (IllegalStateException e) {
   Logging.e(TAG, "initEncodeInternal failed", e);
   release();
   return VideoCodecStatus.FALLBACK_SOFTWARE;
  }

  running = true;
  outputThreadChecker.detachThread();
  outputThread = createOutputThread();
  outputThread.start();

  return VideoCodecStatus.OK;
 }

通过上面初始化可知，摄像头是在自己的surfaceview里面的surface渲染，然后通过opengle共享纹理，将图像通过texture共享给编码器的surface，所以本地能够正常看到自己的图像，但是对方无法看到自己的图像，是由于编码的时候发生错误，而且这种情况只出现在k30手机上

由于发现错误是编码器发生的错误，而且其他型号的小米手机也没出现问题，怀疑有可能是手机厂商对于vp8硬件编码器没有适配好，在谷歌根据错误查找解决方案，暂无发现好的解决方案

发现手机点击开关摄像头，对方能够正常看到视频，偿试用此方案在sdk根据特定机型先关一次再开一次，可以解决问题，但是硬件编码器日志一直在报错，而且每生成一帧都报错，频率非常高，而且单聊切换由于会切换surfureview，也会有很多问题，暂时放弃使用些方法

最后根据特定机型，编码的时候使用软编码，解码还是用硬解码：

```
if("Redmi K30 5G".equals(android.os.Build.MODEL)) {
  encoderFactory = new SoftwareVideoEncoderFactory();
} else {
  encoderFactory = new DefaultVideoEncoderFactory(
          rootEglBase.getEglBaseContext(), true /* enableIntelVp8Encoder */, enableH264HighProfile);
}
decoderFactory = new DefaultVideoDecoderFactory(rootEglBase.getEglBaseContext());
```