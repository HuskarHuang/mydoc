# android webrtc 音视频sdk更改webrtc源码支持H264(基于m79)

应用层创建初始编解码器代码如下：

PeerConnectionClient.java

```java
PeerConnectionFactory factory;
final VideoEncoderFactory encoderFactory;
final VideoDecoderFactory decoderFactory;
encoderFactory = new DefaultVideoEncoderFactory(
        rootEglBase.getEglBaseContext(), true, enableH264HighProfile);
decoderFactory = new DefaultVideoDecoderFactory(rootEglBase.getEglBaseContext());
factory = PeerConnectionFactory.builder()
              .setOptions(options)
              .setAudioDeviceModule(adm)
              .setVideoEncoderFactory(encoderFactory)
              .setVideoDecoderFactory(decoderFactory)
              .createPeerConnectionFactory();
```

从上面可以知道，编解码器相关的配置在DefaultVideoEncoderFactory、DefaultVideoDecoderFactory类中，并且最后通过PeerConnectionFactory将encoderFactory、decoderFactory引用传给c++层，供后续回调使用

## 解码H264支持

sdk\android\api\org\webrtc\DefaultVideoDecoderFactory.java

```java
public class DefaultVideoDecoderFactory implements VideoDecoderFactory {
  private final VideoDecoderFactory hardwareVideoDecoderFactory;
  private final VideoDecoderFactory softwareVideoDecoderFactory = new SoftwareVideoDecoderFactory();
  private final @Nullable VideoDecoderFactory platformSoftwareVideoDecoderFactory;
    
 /**
  * Create decoder factory using default hardware decoder factory.
  */
 public DefaultVideoDecoderFactory(@Nullable EglBase.Context eglContext) {
  this.hardwareVideoDecoderFactory = new HardwareVideoDecoderFactory(eglContext);
  this.platformSoftwareVideoDecoderFactory = new PlatformSoftwareVideoDecoderFactory(eglContext);
 }
 
  @Override
  public @Nullable VideoDecoder createDecoder(VideoCodecInfo codecType) {
    VideoDecoder softwareDecoder = softwareVideoDecoderFactory.createDecoder(codecType);
    final VideoDecoder hardwareDecoder = hardwareVideoDecoderFactory.createDecoder(codecType);
    if (softwareDecoder == null && platformSoftwareVideoDecoderFactory != null) {
      softwareDecoder = platformSoftwareVideoDecoderFactory.createDecoder(codecType);
    }
    if (hardwareDecoder != null && softwareDecoder != null) {
      // Both hardware and software supported, wrap it in a software fallback
      return new VideoDecoderFallback(
          /* fallback= */ softwareDecoder, /* primary= */ hardwareDecoder);
    }
    return hardwareDecoder != null ? hardwareDecoder : softwareDecoder;
      //先判断有硬件解码器，如果有的话，会使用hardwareDecoder，如果没有，会使用softwareDecoder
  }

  @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    LinkedHashSet<VideoCodecInfo> supportedCodecInfos = new LinkedHashSet<VideoCodecInfo>();

    supportedCodecInfos.addAll(Arrays.asList(softwareVideoDecoderFactory.getSupportedCodecs()));
    supportedCodecInfos.addAll(Arrays.asList(hardwareVideoDecoderFactory.getSupportedCodecs()));
    if (platformSoftwareVideoDecoderFactory != null) {
      supportedCodecInfos.addAll(
          Arrays.asList(platformSoftwareVideoDecoderFactory.getSupportedCodecs()));
    }

    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }   
```

sdk\android\api\org\webrtc\VideoDecoderFactory.java 定义如下

```java
public interface VideoDecoderFactory {
 /** Creates a decoder for the given video codec. */
 @Nullable
 @CalledByNative
 default VideoDecoder createDecoder(VideoCodecInfo info) {
  return createDecoder(info.getName());
 }

 /**
  * Enumerates the list of supported video codecs.
  */
 @CalledByNative
 default VideoCodecInfo[] getSupportedCodecs() {
  return new VideoCodecInfo[0];
 }
}
```

可以知道DefaultVideoDecoderFactory实现VideoDecoderFactory,查看上面VideoDecoderFactory的接口定义，可以知道有CalledByNative,说明createDecoder与getSupportedCodecs是由底层c++去调用的

从上面DefaultVideoDecoderFactory的createDecoder可以知道，创建解码器的时候，会先判断有硬件解码器，如果有的话，会使用hardwareDecoder，如果没有，会使用softwareDecoder，getSupportedCodecs会将所有软件、硬件解码相关信息保存起来

上面DefaultVideoDecoderFactory会去new HardwareVideoDecoderFactory,代码如下:

sdk\android\api\org\webrtc\HardwareVideoDecoderFactory.java

```java
public class HardwareVideoDecoderFactory extends MediaCodecVideoDecoderFactory {

 public HardwareVideoDecoderFactory(@Nullable EglBase.Context sharedContext) {
  this(sharedContext, /* codecAllowedPredicate= */ null);
 }

 public HardwareVideoDecoderFactory(@Nullable EglBase.Context sharedContext,
   @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate) {
  super(sharedContext,
    (codecAllowedPredicate == null ? defaultAllowedPredicate
                    : codecAllowedPredicate.and(defaultAllowedPredicate)));
 }

}
```

可以知道最终会调用父类（super）MediaCodecVideoDecoderFactory,而具体的createDecoder与getSupportedCodecs调用都在MediaCodecVideoDecoderFactory中

查看MediaCodecVideoDecoderFactory的createDecoder,如下:

```java
 @Nullable
 @Override
 public VideoDecoder createDecoder(VideoCodecInfo codecType) {
  VideoCodecType type = VideoCodecType.valueOf(codecType.getName());
  MediaCodecInfo info = findCodecForType(type);


  if (info == null) {
   return null;
  }

  CodecCapabilities capabilities = info.getCapabilitiesForType(type.mimeType());
  return new AndroidVideoDecoder(new MediaCodecWrapperFactoryImpl(), info.getName(), type,
    MediaCodecUtils.selectColorFormat(MediaCodecUtils.DECODER_COLOR_FORMATS, capabilities),
    sharedContext);
 }
```

可以看到，createDecoder很简单很自然的就去创建了AndroidVideoDecoder解码，所以这个地方应该不会是我们想要的，毕竟再去看AndroidVideoDecoder类，还是有挺多东西的，在此不做分析；接着往下看getSupportedCodecs

```java
 @Override
 public VideoCodecInfo[] getSupportedCodecs() {
  List<VideoCodecInfo> supportedCodecInfos = new ArrayList<VideoCodecInfo>();
  // Generate a list of supported codecs in order of preference:
  // VP8, VP9, H264 (high profile), and H264 (baseline profile).
  for (VideoCodecType type :
    new VideoCodecType[] {VideoCodecType.VP8, VideoCodecType.VP9, VideoCodecType.H264}) {
   MediaCodecInfo codec = findCodecForType(type);//
   if (codec != null) {
    String name = type.name();
    if (type == VideoCodecType.H264 && isH264HighProfileSupported(codec)) {//
     supportedCodecInfos.add(new VideoCodecInfo(
       name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true)));
    }
    supportedCodecInfos.add(new VideoCodecInfo(
      name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false)));
   }
  }

  return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
 }
```

我们看到了getSupportedCodecs方法，在里面看到了isH264HighProfileSupported方法，看看这方法怎么样：

```java
 private boolean isH264HighProfileSupported(MediaCodecInfo info) {
  String name = info.getName();
  // Support H.264 HP decoding on QCOM chips for Android L and above.
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && name.startsWith(QCOM_PREFIX)) {
   return true;
  }
  // Support H.264 HP decoding on Exynos chips for Android M and above.
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && name.startsWith(EXYNOS_PREFIX)) {
   return true;
  }
  return false;
 }
```

可以看到，官方的注释里面写着H264只支持QCOM（高通）和Exynos芯片，而我们的测试机cpu有的是mtk与麒麟芯片；所以在这个方法当中，索性将最后的return false改成了return true，如下：

```java
 private boolean isH264HighProfileSupported(MediaCodecInfo info) {
  String name = info.getName();
  // Support H.264 HP decoding on QCOM chips for Android L and above.
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && name.startsWith(QCOM_PREFIX)) {
   return true;
  }
  // Support H.264 HP decoding on Exynos chips for Android M and above.
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && name.startsWith(EXYNOS_PREFIX)) {
   return true;
  }
  return true;
 }
```

在getSupportedCodecs方法当中，还看到了findCodecForType方法，而在findCodecForType方法当中又看到了isSupportedCodec方法

```java
 private @Nullable MediaCodecInfo findCodecForType(VideoCodecType type) {
  // HW decoding is not supported on builds before KITKAT.
  if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
   return null;
  }

  for (int i = 0; i < MediaCodecList.getCodecCount(); ++i) {
   MediaCodecInfo info = null;
   try {
    info = MediaCodecList.getCodecInfoAt(i);
   } catch (IllegalArgumentException e) {
    Logging.e(TAG, "Cannot retrieve decoder codec info", e);
   }

   if (info == null || info.isEncoder()) {
    continue;
   }

   if (isSupportedCodec(info, type)) {
    return info;
   }
  }

  return null; // No support for this type.
 }

 // Returns true if the given MediaCodecInfo indicates a supported encoder for the given type.
 private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecType type) {
  String name = info.getName();
  if (!MediaCodecUtils.codecSupportsType(info, type)) {
   return false;
  }
  // Check for a supported color format.
  if (MediaCodecUtils.selectColorFormat(
      MediaCodecUtils.DECODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
    == null) {
   return false;
  }
  return isCodecAllowed(info);
 }

 private boolean isCodecAllowed(MediaCodecInfo info) {
  if (codecAllowedPredicate == null) {
   return true;
  }
  return codecAllowedPredicate.test(info);
 }
```

直接修改isSupportedCodec方法，将最后return的isCodecAllowed方法整个注释掉，然后直接返回true即可，如下：

```java
 private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecType type) {
  String name = info.getName();
  if (!MediaCodecUtils.codecSupportsType(info, type)) {
   return false;
  }
  // Check for a supported color format.
  if (MediaCodecUtils.selectColorFormat(
      MediaCodecUtils.DECODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
    == null) {
   return false;
  }
  return true;
 }
```

到此为止，解码的支持算是改完了；但是在通话之前，webrtc需要交换SDP，SDP在我的理解就是告诉对方，我自己存在着什么实力，解码支持了，但是编码我也要先有这个实力才行

## 编码H264支持

DefaultVideoEncoderFactory会new HardwareVideoEncoderFactory，代码如下：

sdk\android\api\org\webrtc\DefaultVideoEncoderFactory.java

```java
public class DefaultVideoEncoderFactory implements VideoEncoderFactory {
  private final VideoEncoderFactory hardwareVideoEncoderFactory;
  private final VideoEncoderFactory softwareVideoEncoderFactory = new SoftwareVideoEncoderFactory();

  /** Create encoder factory using default hardware encoder factory. */
  public DefaultVideoEncoderFactory(
      EglBase.Context eglContext, boolean enableIntelVp8Encoder, boolean enableH264HighProfile) {
    this.hardwareVideoEncoderFactory =
        new HardwareVideoEncoderFactory(eglContext, enableIntelVp8Encoder, enableH264HighProfile);
  }

  /** Create encoder factory using explicit hardware encoder factory. */
  DefaultVideoEncoderFactory(VideoEncoderFactory hardwareVideoEncoderFactory) {
    this.hardwareVideoEncoderFactory = hardwareVideoEncoderFactory;
  }

  @Nullable
  @Override
  public VideoEncoder createEncoder(VideoCodecInfo info) {
    final VideoEncoder softwareEncoder = softwareVideoEncoderFactory.createEncoder(info);
    final VideoEncoder hardwareEncoder = hardwareVideoEncoderFactory.createEncoder(info);
    if (hardwareEncoder != null && softwareEncoder != null) {
      // Both hardware and software supported, wrap it in a software fallback
      return new VideoEncoderFallback(
          /* fallback= */ softwareEncoder, /* primary= */ hardwareEncoder);
    }
    return hardwareEncoder != null ? hardwareEncoder : softwareEncoder;
  }

  @Override
  public VideoCodecInfo[] getSupportedCodecs() {
    LinkedHashSet<VideoCodecInfo> supportedCodecInfos = new LinkedHashSet<VideoCodecInfo>();

    supportedCodecInfos.addAll(Arrays.asList(softwareVideoEncoderFactory.getSupportedCodecs()));
    supportedCodecInfos.addAll(Arrays.asList(hardwareVideoEncoderFactory.getSupportedCodecs()));

    return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
  }
}
```

可以知道DefaultVideoEncoderFactory实现VideoEncoderFactory,查看VideoEncoderFactory定义

```java
public interface VideoEncoderFactory {
 @Nullable @CalledByNative VideoEncoder createEncoder(VideoCodecInfo info);

 @CalledByNative VideoCodecInfo[] getSupportedCodecs();

 @CalledByNative
 default VideoCodecInfo[] getImplementations() {
  return getSupportedCodecs();
 }
}
```

可以知道DefaultVideoEncoderFactory实现VideoEncoderFactory,查看上面VideoEncoderFactory的接口定义，可以知道有CalledByNative,说明createEncoder与getSupportedCodecs是由底层c++去调用的

从上面DefaultVideoEncoderFactory的createDecoder可以知道，创建编码器的时候，会先判断有硬件编码器，如果有的话，会使用hardwareEncoder，如果没有，会使用softwareEncoder，getSupportedCodecs会将所有软件、硬件编码相关信息保存起来

下面是HardwareVideoEncoderFactory相关构造方法

sdk\android\api\org\webrtc\HardwareVideoEncoderFactory.java

```java
public class HardwareVideoEncoderFactory implements VideoEncoderFactory {

 public HardwareVideoEncoderFactory(
   EglBase.Context sharedContext, boolean enableIntelVp8Encoder, boolean enableH264HighProfile) {
  this(sharedContext, enableIntelVp8Encoder, enableH264HighProfile,
    /* codecAllowedPredicate= */ null);
 }

 public HardwareVideoEncoderFactory(EglBase.Context sharedContext, boolean enableIntelVp8Encoder,
   boolean enableH264HighProfile, @Nullable Predicate<MediaCodecInfo> codecAllowedPredicate) {
  // Texture mode requires EglBase14.
  if (sharedContext instanceof EglBase14.Context) {
   this.sharedContext = (EglBase14.Context) sharedContext;
  } else {
   Logging.w(TAG, "No shared EglBase.Context. Encoders will not use texture mode.");
   this.sharedContext = null;
  }
  this.enableIntelVp8Encoder = enableIntelVp8Encoder;
  this.enableH264HighProfile = enableH264HighProfile;
  this.codecAllowedPredicate = codecAllowedPredicate;
 }
```

从上面可以知道，如果要支持h264，需要将enableIntelVp8Encoder设置为false，详细可以看

```java
  private boolean isHardwareSupportedInCurrentSdkVp8(MediaCodecInfo info) {
    String name = info.getName();
    // QCOM Vp8 encoder is supported in KITKAT or later.
    return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
        // Exynos VP8 encoder is supported in M or later.
        || (name.startsWith(EXYNOS_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M)
        // Intel Vp8 encoder is supported in LOLLIPOP or later, with the intel encoder enabled.
        || (name.startsWith(INTEL_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP
               && enableIntelVp8Encoder);
  }
```

即上层要改为

```java
encoderFactory = new DefaultVideoEncoderFactory(
        rootEglBase.getEglBaseContext(), false, enableH264HighProfile);
```

下面看HardwareVideoEncoderFactory对createEncoder与getSupportedCodecs的实现

```java
 @Nullable
 @Override
 public VideoEncoder createEncoder(VideoCodecInfo input) {
  // HW encoding is not supported below Android Kitkat.
  if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
   return null;
  }

  VideoCodecType type = VideoCodecType.valueOf(input.name);
  MediaCodecInfo info = findCodecForType(type);

  if (info == null) {
   return null;
  }

  String codecName = info.getName();
  String mime = type.mimeType();
  Integer surfaceColorFormat = MediaCodecUtils.selectColorFormat(
    MediaCodecUtils.TEXTURE_COLOR_FORMATS, info.getCapabilitiesForType(mime));
  Integer yuvColorFormat = MediaCodecUtils.selectColorFormat(
    MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(mime));

  if (type == VideoCodecType.H264) {
   boolean isHighProfile = H264Utils.isSameH264Profile(
     input.params, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true));
   boolean isBaselineProfile = H264Utils.isSameH264Profile(
     input.params, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false));

   if (!isHighProfile && !isBaselineProfile) {
    return null;
   }
   if (isHighProfile && !isH264HighProfileSupported(info)) {
    return null;
   }
  }

  return new HardwareVideoEncoder(new MediaCodecWrapperFactoryImpl(), codecName, type,
    surfaceColorFormat, yuvColorFormat, input.params, getKeyFrameIntervalSec(type),
    getForcedKeyFrameIntervalMs(type, codecName), createBitrateAdjuster(type, codecName),
    sharedContext);
 }

 @Override
 public VideoCodecInfo[] getSupportedCodecs() {
  // HW encoding is not supported below Android Kitkat.
  if (Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
   return new VideoCodecInfo[0];
  }

  List<VideoCodecInfo> supportedCodecInfos = new ArrayList<VideoCodecInfo>();
  // Generate a list of supported codecs in order of preference:
  // VP8, VP9, H264 (high profile), and H264 (baseline profile).
  for (VideoCodecType type :
    new VideoCodecType[] {VideoCodecType.VP8, VideoCodecType.VP9, VideoCodecType.H264}) {
   MediaCodecInfo codec = findCodecForType(type);
   if (codec != null) {
    String name = type.name();
    // TODO(sakal): Always add H264 HP once WebRTC correctly removes codecs that are not
    // supported by the decoder.
    if (type == VideoCodecType.H264 && isH264HighProfileSupported(codec)) {
     supportedCodecInfos.add(new VideoCodecInfo(
       name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ true)));
    }

    supportedCodecInfos.add(new VideoCodecInfo(
      name, MediaCodecUtils.getCodecProperties(type, /* highProfile= */ false)));
   }
  }

  return supportedCodecInfos.toArray(new VideoCodecInfo[supportedCodecInfos.size()]);
 }

 private @Nullable MediaCodecInfo findCodecForType(VideoCodecType type) {
  for (int i = 0; i < MediaCodecList.getCodecCount(); ++i) {
   MediaCodecInfo info = null;
   try {
    info = MediaCodecList.getCodecInfoAt(i);
   } catch (IllegalArgumentException e) {
    Logging.e(TAG, "Cannot retrieve encoder codec info", e);
   }



   if (info == null || !info.isEncoder()) {
    continue;
   }

   if (isSupportedCodec(info, type)) {
    return info;
   }
  }
  return null; // No support for this type.
 }

 private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecType type) {
  if (!MediaCodecUtils.codecSupportsType(info, type)) {
   return false;
  }
  // Check for a supported color format.
  if (MediaCodecUtils.selectColorFormat(
      MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
    == null) {
   return false;
  }
  return isHardwareSupportedInCurrentSdk(info, type) && isMediaCodecAllowed(info);
 }

 private boolean isHardwareSupportedInCurrentSdk(MediaCodecInfo info, VideoCodecType type) {
  switch (type) {
   case VP8:
    return isHardwareSupportedInCurrentSdkVp8(info);
   case VP9:
    return isHardwareSupportedInCurrentSdkVp9(info);
   case H264:
    return isHardwareSupportedInCurrentSdkH264(info);
  }
  return false;
 }

 private boolean isHardwareSupportedInCurrentSdkH264(MediaCodecInfo info) {
  // First, H264 hardware might perform poorly on this model.
  if (H264_HW_EXCEPTION_MODELS.contains(Build.MODEL)) {
   return false;
  }
  String name = info.getName();
  // QCOM H264 encoder is supported in KITKAT or later.
  return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
    // Exynos H264 encoder is supported in LOLLIPOP or later.
    || (name.startsWith(EXYNOS_PREFIX)
        && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP);
 }
```

和解码同样的道理，我们在该类当中也能找到getSupportedCodecs和findCodecForType方法，对这两个方法查看和跟踪能看到isH264HighProfileSupported方法和isHardwareSupportedInCurrentSdkH264方法，这两个方法的修改如下：

```java
 private boolean isH264HighProfileSupported(MediaCodecInfo info) {
 //  return enableH264HighProfile && Build.VERSION.SDK_INT > Build.VERSION_CODES.M
 //    && info.getName().startsWith(EXYNOS_PREFIX);

   retrun enableH264HighProfile ;
 }

 private boolean isHardwareSupportedInCurrentSdkH264(MediaCodecInfo info) {
  // First, H264 hardware might perform poorly on this model.
  // if (H264_HW_EXCEPTION_MODELS.contains(Build.MODEL)) {
  //  return false;
  // }
  // String name = info.getName();
  // // QCOM H264 encoder is supported in KITKAT or later.
  // return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
  //   // Exynos H264 encoder is supported in LOLLIPOP or later.
  //   || (name.startsWith(EXYNOS_PREFIX)
  //      && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP);
  return true;
 }
```

改完之后，重新打包编译

```shell
➜  src git:(m79) ✗ pwd
/home/hwh/webrtc/webrtc_android/src
➜  src git:(m79) ✗ tools_webrtc/android/build_aar.py --build-dir out --arch "armeabi-v7a" "arm64-v8a" "x86" "x86_64"
INFO:root:Building: armeabi-v7a
Done. Made 3659 targets from 269 files in 1167ms
ninja: Entering directory `out/armeabi-v7a'
ninja: no work to do.
INFO:root:Building: arm64-v8a
Done. Made 3607 targets from 267 files in 1724ms
ninja: Entering directory `out/arm64-v8a'
ninja: no work to do.
INFO:root:Building: x86
Done. Made 3635 targets from 268 files in 791ms
ninja: Entering directory `out/x86'
ninja: no work to do.
INFO:root:Building: x86_64
Done. Made 3635 targets from 268 files in 884ms
ninja: Entering directory `out/x86_64'
ninja: no work to do.
INFO:root:Collecting common files.
INFO:root:Collecting: armeabi-v7a
INFO:root:Collecting: arm64-v8a
INFO:root:Collecting: x86
INFO:root:Collecting: x86_64
INFO:root:List of licenses: webrtc, abseil-cpp, android_deps:com_android_support_support_annotations.*, android_ndk, android_sdk, auto, base64, bazel, boringssl, errorprone, fft, fft4g, fiat, g711, g722, guava, ijar, libc++, libc++abi, libevent, libjpeg_turbo, libsrtp, libvpx, libyuv, opus, pffft, protobuf, rnnoise, sigslot, spl_sqrt_floor, usrsctp, yasm, zlib
INFO:root:Skipping compile time or internal dependency: android_deps:com_android_support_support_annotations.*
INFO:root:Skipping compile time or internal dependency: yasm
➜  src git:(m79) ✗ 

```