# android webrtc wifi切4g有些手机会出现视频画面卡死解决方案

问题描述：进行wifi切4g的时候，当与对方进行视频通话的时候，自己的手机的画面显示正常，但是对方的手机显示自己的画面却处于切换之前的最后一帧，且持续2到3分钟，画面才恢复正常，但是自己手上的小米8手机却只是在切换的时候卡顿一下，几秒后可以恢复正常。

首先在对方的手机做一个每10秒钟刷新一次，发现还是一样，问题没有解决，而且通过流量软件，发现对方也是有接收到码流的下行流量，由此可以确定问题不是播放端的问题

由此可以定位，问题是出现在发送端的，通过服务器录制发送上去的视频流，然后进行播放，发现切换网络后，在对方显示最后卡顿一帧的时候，后面都出现花屏了，由此发现，应该是i帧的间隔太长了导致了

查看webrtc源码

src/java/org/webrtc/HardwareVideoEncoder.java

MediaFormat format = MediaFormat.createVideoFormat(codecType.mimeType(), width, height);
format.setInteger(MediaFormat.KEY_BIT_RATE, adjustedBitrate);
format.setInteger(KEY_BITRATE_MODE, VIDEO_ControlRateConstant);
format.setInteger(MediaFormat.KEY_COLOR_FORMAT, colorFormat);
format.setInteger(MediaFormat.KEY_FRAME_RATE, bitrateAdjuster.getCodecConfigFramerate());
**Log.w(TAG, "keyFrameIntervalSec = " + keyFrameIntervalSec);**
format.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, keyFrameIntervalSec);



接入小米8手机，发现打印的是100，即此处代码是进行硬件编码器的初始化，并且初始i帧的间隔为100帧，而且帧率为15~20，即最长大约5秒能够重新生成一个i帧，后续页面就可以正常

但是接入其他手机有问题的时候，发现这行打印没有，因此出现问题时候，应该使用的是软编码，即webrtc会首先查找是否有对应的硬件编码器，如果找不到，就使用软件编码器

通过分析查找webrtc底层软编码的位置，如下：

sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java

public class SoftwareVideoEncoderFactory implements VideoEncoderFactory {
@Nullable
@Override
**public VideoEncoder createEncoder(VideoCodecInfo info) {**
**if ([info.name](http://info.name).equalsIgnoreCase("VP8")) {**
**return new LibvpxVp8Encoder();**
**}**
if ([info.name](http://info.name).equalsIgnoreCase("VP9") && LibvpxVp9Encoder.nativeIsSupported()) {
return new LibvpxVp9Encoder();
}

return null;
}



即会new一个LibvpxVp8Encoder的java类，然后如下代码调用**createNativeVideoEncoder**

src/sdk/android/api/org/webrtc/LibvpxVp8Encoder.java

public class LibvpxVp8Encoder extends WrappedNativeVideoEncoder {
**@Override**
**public long createNativeVideoEncoder() {**
**return nativeCreateEncoder();**
**}**

**static native long nativeCreateEncoder();**

@Override
public boolean isHardwareEncoder() {
return false;
}
}

会调用**nativeCreateEncoder**会调用到**JNI_LibvpxVp8Encoder_CreateEncoder**，代码如下：

sdk/android/src/jni/[vp8_codec.cc](http://vp8_codec.cc)

\#include <jni.h>

\#include "modules/video_coding/codecs/vp8/include/vp8.h"
\#include "sdk/android/generated_libvpx_vp8_jni/LibvpxVp8Decoder_jni.h"
\#include "sdk/android/generated_libvpx_vp8_jni/LibvpxVp8Encoder_jni.h"
\#include "sdk/android/src/jni/jni_helpers.h"

namespace webrtc {
namespace jni {

**static jlong JNI_LibvpxVp8Encoder_CreateEncoder(JNIEnv* jni) {**
**return jlongFromPointer(VP8Encoder::Create(nullptr).release());**
**}**

static jlong JNI_LibvpxVp8Decoder_CreateDecoder(JNIEnv* jni) {
return jlongFromPointer(VP8Decoder::Create().release());
}

} // namespace jni
} // namespace webrtc

而**VP8Encoder::Create(nullptr)**会调用

modules/video_coding/codecs/vp8/[libvpx_vp8_encoder.cc](http://libvpx_vp8_encoder.cc)

std::unique_ptr<VideoEncoder> VP8Encoder::Create(
std::unique_ptr<Vp8FrameBufferControllerFactory>
frame_buffer_controller_factory) {
return std::make_unique<**LibvpxVp8Encoder**>(
std::move(frame_buffer_controller_factory));
}

即创建一个**LibvpxVp8Encoder**的c++的类

然后调用**LibvpxVp8Encoder**的**InitEncode**，里面会设置i帧的帧间距，即如下代码**inst->VP8().keyFrameInterval**

src/modules/video_coding/codecs/vp8/[libvpx_vp8_encoder.cc](http://libvpx_vp8_encoder.cc)

int LibvpxVp8Encoder::**InitEncode**(const VideoCodec* inst,
const VideoEncoder::Settings& settings) {

//hwh add
**RTC_LOG(LS_WARNING) << "== LibvpxVp8Encoder::InitEncode keyFrameInterval = " << (inst->VP8().keyFrameInterval) ;**
//hwh add end
if (inst->VP8().keyFrameInterval > 0) {
vpx_configs_[0].kf_mode = VPX_KF_AUTO;
vpx_configs_[0].kf_max_dist = inst->VP8().keyFrameInterval;
} else {
vpx_configs_[0].kf_mode = VPX_KF_DISABLED;
}



发现有问题的手机会有打印，并且**keyFrameInterval = 3000；**

查找keyFrameInterval的定义如下：

api/video_codecs/[video_encoder.cc](http://video_encoder.cc)

namespace webrtc {

// TODO(mflodman): Add default complexity for VP9 and VP9.
VideoCodecVP8 VideoEncoder::GetDefaultVp8Settings() {
VideoCodecVP8 vp8_settings;
memset(&vp8_settings, 0, sizeof(vp8_settings));

vp8_settings.numberOfTemporalLayers = 1;
vp8_settings.denoisingOn = true;
vp8_settings.automaticResizeOn = false;
vp8_settings.frameDroppingOn = true;
**vp8_settings.keyFrameInterval = 3000;**

return vp8_settings;
}

VideoCodecVP9 VideoEncoder::GetDefaultVp9Settings() {
VideoCodecVP9 vp9_settings;
memset(&vp9_settings, 0, sizeof(vp9_settings));

vp9_settings.numberOfTemporalLayers = 1;
vp9_settings.denoisingOn = true;
vp9_settings.frameDroppingOn = true;
**vp9_settings.keyFrameInterval = 3000;**
vp9_settings.adaptiveQpMode = true;
vp9_settings.automaticResizeOn = true;
vp9_settings.numberOfSpatialLayers = 1;
vp9_settings.flexibleMode = false;
vp9_settings.interLayerPred = InterLayerPredMode::kOn;

return vp9_settings;
}

VideoCodecH264 VideoEncoder::GetDefaultH264Settings() {
VideoCodecH264 h264_settings;
memset(&h264_settings, 0, sizeof(h264_settings));

h264_settings.frameDroppingOn = true;
**h264_settings.keyFrameInterval = 3000;**
h264_settings.numberOfTemporalLayers = 1;

return h264_settings;
}

**即每3000帧才产生一个i帧，即按15~20帧率来计算的话，差不多在3分多钟才产生一个i帧，这也就是为什么对方要隔个几分钟后才能恢复正常的原因，将此值改为300后，后面就正常了**