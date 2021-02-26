# android webrtc 音视频支持h264软编软解

由于webrtc默认只支持vp8、vp9硬编、硬解，软编、软解，对H264目前只支持硬编、硬解，所以如果当硬编、硬解出现问题之后，由于java层没有支持软编、软解，所以没有办法跳到软编、软解，这对sdk的兼容性不完善，最简单的就是目前手头上的展讯芯片手机与模拟器，添加之后可以解决手机不能编码的问题，对方能够正常看到图像了。

下面从java层到底层，对webrtc添加H264软编、软解

### 添加java、与jni支持

#### 添加LibH264Decoder与LibH264Encoder类

参照sdk/android/api/org/webrtc/LibvpxVp8Encoder.java、sdk/android/api/org/webrtc/LibvpxVp8Decoder.java创建LibH264Decoder与LibH264Encoder类

如下：

```java
/*
 *  Copyright (c) 2017 The WebRTC project authors. All Rights Reserved.
 *
 *  Use of this source code is governed by a BSD-style license
 *  that can be found in the LICENSE file in the root of the source
 *  tree. An additional intellectual property rights grant can be found
 *  in the file PATENTS.  All contributing project authors may
 *  be found in the AUTHORS file in the root of the source tree.
 */

package org.webrtc;

public class LibH264Decoder extends WrappedNativeVideoDecoder {
  @Override
  public long createNativeVideoDecoder() {
    return nativeCreateDecoder();
  }

  static native long nativeCreateDecoder();
}
```

```java
/*
 *  Copyright (c) 2017 The WebRTC project authors. All Rights Reserved.
 *
 *  Use of this source code is governed by a BSD-style license
 *  that can be found in the LICENSE file in the root of the source
 *  tree. An additional intellectual property rights grant can be found
 *  in the file PATENTS.  All contributing project authors may
 *  be found in the AUTHORS file in the root of the source tree.
 */

package org.webrtc;

public class LibH264Encoder extends WrappedNativeVideoEncoder {
  @Override
  public long createNativeVideoEncoder() {
    return nativeCreateEncoder();
  }

  static native long nativeCreateEncoder();

  @Override
  public boolean isHardwareEncoder() {
    return false;
  }
}
```

#### SoftwareVideoDecoderFactory添加支持

```java
diff --git a/sdk/android/api/org/webrtc/SoftwareVideoDecoderFactory.java b/sdk/android/api/org/webrtc/SoftwareVideoDecoderFactory.java
index 7abe3505a6..5498655593 100644
--- a/sdk/android/api/org/webrtc/SoftwareVideoDecoderFactory.java
+++ b/sdk/android/api/org/webrtc/SoftwareVideoDecoderFactory.java
@@ -26,6 +26,9 @@ public class SoftwareVideoDecoderFactory implements VideoDecoderFactory {
   @Nullable
   @Override
   public VideoDecoder createDecoder(VideoCodecInfo codecType) {
+    if (codecType.getName().equalsIgnoreCase("H264")) {
+      return new LibH264Decoder();
+    }
     if (codecType.getName().equalsIgnoreCase("VP8")) {
       return new LibvpxVp8Decoder();
     }
@@ -44,6 +47,7 @@ public class SoftwareVideoDecoderFactory implements VideoDecoderFactory {
   static VideoCodecInfo[] supportedCodecs() {
     List<VideoCodecInfo> codecs = new ArrayList<VideoCodecInfo>();
 
+    codecs.add(new VideoCodecInfo("H264", new HashMap<>()));
     codecs.add(new VideoCodecInfo("VP8", new HashMap<>()));
     if (LibvpxVp9Decoder.nativeIsSupported()) {
       codecs.add(new VideoCodecInfo("VP9", new HashMap<>()));

```

#### SoftwareVideoEncoderFactory添加支持

```java
diff --git a/sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java b/sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java
index ed70d228ef..5d61eb8ee9 100644
--- a/sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java
+++ b/sdk/android/api/org/webrtc/SoftwareVideoEncoderFactory.java
@@ -19,6 +19,9 @@ public class SoftwareVideoEncoderFactory implements VideoEncoderFactory {
   @Nullable
   @Override
   public VideoEncoder createEncoder(VideoCodecInfo info) {
+    if (info.name.equalsIgnoreCase("H264")) {
+      return new LibH264Encoder();
+    }
     if (info.name.equalsIgnoreCase("VP8")) {
       return new LibvpxVp8Encoder();
     }
@@ -37,11 +40,13 @@ public class SoftwareVideoEncoderFactory implements VideoEncoderFactory {
   static VideoCodecInfo[] supportedCodecs() {
     List<VideoCodecInfo> codecs = new ArrayList<VideoCodecInfo>();
 
+//    codecs.add(new VideoCodecInfo("H264", new HashMap<>()));
+    codecs.add(new VideoCodecInfo("H264", MediaCodecUtils.getCodecProperties(VideoCodecType.H264, /* highProfile= */ true)));
+    codecs.add(new VideoCodecInfo("H264", MediaCodecUtils.getCodecProperties(VideoCodecType.H264, /* highProfile= */ false)));

     codecs.add(new VideoCodecInfo("VP8", new HashMap<>()));
     if (LibvpxVp9Encoder.nativeIsSupported()) {
       codecs.add(new VideoCodecInfo("VP9", new HashMap<>()));
     }
 
     return codecs.toArray(new VideoCodecInfo[codecs.size()]);
   }
 }

```

#### LibH264Encoder与LibH264Decoder底层jni支持

LibH264Encoder.java会通过nativeCreateEncoder调用到底层c++，LibH264Decoder.java会通过nativeCreateDecoder调用到底层c++，模仿sdk/android/src/jni/vp8_codec.cc创建一个h264_codec.cc并且更新对应的.h引用，如下：

```c++
/*
 *  Copyright 2017 The WebRTC project authors. All Rights Reserved.
 *
 *  Use of this source code is governed by a BSD-style license
 *  that can be found in the LICENSE file in the root of the source
 *  tree. An additional intellectual property rights grant can be found
 *  in the file PATENTS.  All contributing project authors may
 *  be found in the AUTHORS file in the root of the source tree.
 */

#include <jni.h>

#include "modules/video_coding/codecs/h264/include/h264.h"
#include "sdk/android/generated_lib_h264_jni/LibH264Decoder_jni.h"
#include "sdk/android/generated_lib_h264_jni/LibH264Encoder_jni.h"
#include "sdk/android/src/jni/jni_helpers.h"

namespace webrtc {
namespace jni {

static jlong JNI_LibH264Encoder_CreateEncoder(JNIEnv* jni) {
  return jlongFromPointer(H264Encoder::Create(cricket::VideoCodec(cricket::kH264CodecName)).release());
}

static jlong JNI_LibH264Decoder_CreateDecoder(JNIEnv* jni) {
  return jlongFromPointer(H264Decoder::Create().release());
}

}  // namespace jni
}  // namespace webrtc

```

#### BUILD.gn添加支持

由于webrtc源码是通过.gn文件组织编译的，需要将新加入的文件添加进去，更改如下：

```
diff --git a/sdk/android/BUILD.gn b/sdk/android/BUILD.gn
index f6fd46230c..9c58fbc29d 100644
--- a/sdk/android/BUILD.gn
+++ b/sdk/android/BUILD.gn
@@ -47,6 +47,7 @@ if (is_android) {
       ":libjingle_peerconnection_metrics_default_java",
       ":libvpx_vp8_java",
       ":libvpx_vp9_java",
+      ":lib_h264_java",
       ":logging_java",
       ":peerconnection_java",
       ":screencapturer_java",
@@ -504,6 +505,20 @@ if (is_android) {
     ]
   }
 
+  rtc_android_library("lib_h264_java") {
+    visibility = [ "*" ]
+    java_files = [
+      "api/org/webrtc/LibH264Decoder.java",
+      "api/org/webrtc/LibH264Encoder.java",
+    ]
+    deps = [
+      ":base_java",
+      ":video_api_java",
+      ":video_java",
+      "//rtc_base:base_java",
+    ]
+  }
+
   rtc_android_library("swcodecs_java") {
     visibility = [ "*" ]
     java_files = [
@@ -515,6 +530,7 @@ if (is_android) {
       ":base_java",
       ":libvpx_vp8_java",
       ":libvpx_vp9_java",
+      ":lib_h264_java",
       ":video_api_java",
       ":video_java",
       "//rtc_base:base_java",
@@ -838,12 +854,27 @@ if (current_os == "linux" || is_android) {
     ]
   }
 
+    rtc_static_library("lib_h264_jni") {
+    visibility = [ "*" ]
+    allow_poison = [ "software_video_codecs" ]
+    sources = [
+      "src/jni/h264_codec.cc",
+    ]
+    deps = [
+      ":base_jni",
+      ":generated_lib_h264_jni",
+      ":video_jni",
+      "../../modules/video_coding:webrtc_h264",
+    ]
+  }
+
   rtc_static_library("swcodecs_jni") {
     visibility = [ "*" ]
     allow_poison = [ "software_video_codecs" ]
     deps = [
       ":libvpx_vp8_jni",
       ":libvpx_vp9_jni",
+      ":lib_h264_jni",
     ]
   }
 
@@ -1262,6 +1293,16 @@ if (current_os == "linux" || is_android) {
     jni_generator_include = "//sdk/android/src/jni/jni_generator_helper.h"
   }
 
+  generate_jni("generated_lib_h264_jni") {
+    sources = [
+      "api/org/webrtc/LibH264Decoder.java",
+      "api/org/webrtc/LibH264Encoder.java",
+    ]
+
+    namespace = "webrtc::jni"
+    jni_generator_include = "//sdk/android/src/jni/jni_generator_helper.h"
+  }
+
   generate_jni("generated_peerconnection_jni") {
     sources = [
       "api/org/webrtc/AudioTrack.java",

```

### ffmpeg软解支持

通过h264_codec.cc可以知道，会分别创建H264Encoder::Create(cricket::VideoCodec(cricket::kH264CodecName))与H264Decoder::Create()，分析跟踪可以知道，编码使用的是openh264，默认编译已经支持，解码使用的是ffmpeg，并且依赖于宏定义WEBRTC_USE_H264，而且此宏依赖于rtc_use_h264，而且ffmpeg默认没有打开h264解码器，下面为ffmpeg软解的支持

#### 修改config文件支持

修改文件如下：

third_party/ffmpeg/chromium/config/Chrome/android/arm-neon/config.h

third_party/ffmpeg/chromium/config/Chrome/android/arm64/config.h

third_party/ffmpeg/chromium/config/Chrome/android/ia32/config.h

third_party/ffmpeg/chromium/config/Chrome/android/x64/config.h

修改如下：

```c
--- a/chromium/config/Chrome/android/arm-neon/config.h
+++ b/chromium/config/Chrome/android/arm-neon/config.h
@@ -794,7 +794,8 @@
 #define CONFIG_H263I_DECODER 0
 #define CONFIG_H263P_DECODER 0
 #define CONFIG_H263_V4L2M2M_DECODER 0
-#define CONFIG_H264_DECODER 0
+// #define CONFIG_H264_DECODER 0 hwh add
+#define CONFIG_H264_DECODER 1
 #define CONFIG_H264_CRYSTALHD_DECODER 0
 #define CONFIG_H264_V4L2M2M_DECODER 0
 #define CONFIG_H264_MEDIACODEC_DECODER 0

```

#### 修改parser_list.c与codec_list.c支持

修改文件如下：

third_party/ffmpeg/chromium/config/Chrome/android/arm-neon/libavcodec/codec_list.c

third_party/ffmpeg/chromium/config/Chrome/android/arm-neon/libavcodec/parser_list.c

third_party/ffmpeg/chromium/config/Chrome/android/arm64/libavcodec/codec_list.c

third_party/ffmpeg/chromium/config/Chrome/android/arm64/libavcodec/parser_list.c

third_party/ffmpeg/chromium/config/Chrome/android/ia32/libavcodec/codec_list.c

third_party/ffmpeg/chromium/config/Chrome/android/ia32/libavcodec/parser_list.c

third_party/ffmpeg/chromium/config/Chrome/android/x64/libavcodec/codec_list.c

third_party/ffmpeg/chromium/config/Chrome/android/x64/libavcodec/parser_list.c

```c
--- a/chromium/config/Chrome/android/arm-neon/libavcodec/codec_list.c
+++ b/chromium/config/Chrome/android/arm-neon/libavcodec/codec_list.c
@@ -13,4 +13,6 @@ static const AVCodec * const codec_list[] = {
     &ff_pcm_s32le_decoder,
     &ff_pcm_u8_decoder,
     &ff_libopus_decoder,
+    // hwh add
+    &ff_h264_decoder,
     NULL };
```

```c
--- a/chromium/config/Chrome/android/arm-neon/libavcodec/parser_list.c
+++ b/chromium/config/Chrome/android/arm-neon/libavcodec/parser_list.c
@@ -5,4 +5,6 @@ static const AVCodecParser * const parser_list[] = {
     &ff_opus_parser,
     &ff_vorbis_parser,
     &ff_vp9_parser,
+    //hwh add
+    &ff_h264_parser,
     NULL };
```

#### 修改ffmpeg_generated.gni将相关文件编译进去

主要加入is_android、 current_cpu=="x64"、current_cpu == "x86"、current_cpu == "arm"、current_cpu == "arm64"相关支持

```
diff --git a/ffmpeg_generated.gni b/ffmpeg_generated.gni
index 5d89a723f5..b78d1f2b06 100644
--- a/ffmpeg_generated.gni
+++ b/ffmpeg_generated.gni
@@ -190,7 +190,7 @@ if ((is_android && current_cpu == "arm" && arm_use_neon) || (is_android && curre
   ]
 }
 
-if ((is_mac) || (is_win) || (use_linux_config)) {
+if ((is_mac) || (is_win) || (use_linux_config) || (is_android)) {
   ffmpeg_c_sources += [
     "libavcodec/autorename_libavcodec_hpeldsp.c",
     "libavcodec/autorename_libavcodec_videodsp.c",
@@ -227,7 +227,7 @@ if ((current_cpu == "x64" && ffmpeg_branding == "Chrome") || (is_android && curr
   ]
 }
 
-if ((is_mac && ffmpeg_branding == "Chrome") || (is_win && ffmpeg_branding == "Chrome") || (use_linux_config && ffmpeg_branding == "Chrome") || (use_linux_config && ffmpeg_branding == "ChromeOS")) {
+if ((is_mac && ffmpeg_branding == "Chrome") || (is_win && ffmpeg_branding == "Chrome") || (use_linux_config && ffmpeg_branding == "Chrome") || (use_linux_config && ffmpeg_branding == "ChromeOS") ||  (is_android)) {
   ffmpeg_c_sources += [
     "libavcodec/cabac.c",
     "libavcodec/h2645_parse.c",
@@ -311,7 +311,7 @@ if ((is_android && current_cpu == "x64") || (is_mac) || (is_win && current_cpu =
   ]
 }
 
-if ((is_mac) || (is_win && current_cpu == "x64") || (is_win && current_cpu == "x86") || (use_linux_config && current_cpu == "x64") || (use_linux_config && current_cpu == "x86")) {
+if ((is_mac) || (is_win && current_cpu == "x64") || (is_win && current_cpu == "x86") || (use_linux_config && current_cpu == "x64") || (use_linux_config && current_cpu == "x86") || (is_android && current_cpu=="x64") || (is_android && current_cpu == "x86")) {
   ffmpeg_c_sources += [
     "libavcodec/x86/autorename_libavcodec_x86_videodsp_init.c",
     "libavcodec/x86/h264_intrapred_init.c",
@@ -407,7 +407,7 @@ if (use_linux_config && ffmpeg_branding == "ChromeOS") {
   ]
 }
 
-if ((use_linux_config && current_cpu == "arm" && arm_use_neon) || (use_linux_config && current_cpu == "arm")) {
+if ((use_linux_config && current_cpu == "arm" && arm_use_neon) || (use_linux_config && current_cpu == "arm") || (is_android && current_cpu == "arm")) {
   ffmpeg_c_sources += [
     "libavcodec/arm/h264pred_init_arm.c",
     "libavcodec/arm/hpeldsp_init_arm.c",
@@ -427,7 +427,7 @@ if ((use_linux_config && current_cpu == "arm" && arm_use_neon) || (use_linux_con
   ]
 }
 
-if ((use_linux_config && current_cpu == "mips64el") || (use_linux_config && current_cpu == "mipsel")) {
+if ((use_linux_config && current_cpu == "mips64el") || (use_linux_config && current_cpu == "mipsel") || (is_android && current_cpu == "mips64el")) {
   ffmpeg_c_sources += [
     "libavcodec/mips/autorename_libavcodec_mips_videodsp_init.c",
     "libavcodec/mips/h264pred_init_mips.c",
@@ -446,7 +446,7 @@ if ((is_android && current_cpu == "mips64el" && ffmpeg_branding == "Chrome") ||
   ]
 }
 
-if ((is_android && current_cpu == "arm64") || (is_win && current_cpu == "arm64") || (use_linux_config && current_cpu == "arm64")) {
+if ((is_android && current_cpu == "arm64") || (is_win && current_cpu == "arm64") || (use_linux_config && current_cpu == "arm64") || (is_android && current_cpu == "arm64")) {
   ffmpeg_c_sources += [
     "libavcodec/aarch64/fft_init_aarch64.c",
     "libavcodec/aarch64/mpegaudiodsp_init.c",
@@ -463,7 +463,7 @@ if ((is_android && current_cpu == "arm64") || (is_win && current_cpu == "arm64")
   ]
 }
 
-if ((is_win && current_cpu == "arm64") || (use_linux_config && current_cpu == "arm64")) {
+if ((is_win && current_cpu == "arm64") || (use_linux_config && current_cpu == "arm64") || (is_android && current_cpu =="arm64")) {
   ffmpeg_c_sources += [
     "libavcodec/aarch64/h264pred_init.c",
     "libavcodec/aarch64/hpeldsp_init_aarch64.c",
@@ -511,7 +511,7 @@ if ((is_android && current_cpu == "arm" && arm_use_neon) || (use_linux_config &&
   ]
 }
 
-if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "arm" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && ffmpeg_branding == "ChromeOS")) {
+if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "arm" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && ffmpeg_branding == "ChromeOS") || (is_android && current_cpu =="arm" && arm_use_neon)) {
   ffmpeg_c_sources += [
     "libavcodec/arm/h264chroma_init_arm.c",
     "libavcodec/arm/h264dsp_init_arm.c",
@@ -522,7 +522,7 @@ if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding
   ]
 }
 
-if ((is_mac && ffmpeg_branding == "Chrome") || (is_win && current_cpu == "x64" && ffmpeg_branding == "Chrome") || (is_win && current_cpu == "x86" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x64" && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "x86" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x86" && ffmpeg_branding == "ChromeOS")) {
+if ((is_mac && ffmpeg_branding == "Chrome") || (is_win && current_cpu == "x64" && ffmpeg_branding == "Chrome") || (is_win && current_cpu == "x86" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x64" && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "x86" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "x86" && ffmpeg_branding == "ChromeOS") || (is_android && current_cpu == "x64") || (is_android && current_cpu == "x86")) {
   ffmpeg_c_sources += [
     "libavcodec/x86/h264_qpel.c",
     "libavcodec/x86/h264chroma_init.c",
@@ -550,7 +550,7 @@ if ((is_android && current_cpu == "mips64el") || (use_linux_config && current_cp
   ]
 }
 
-if ((use_linux_config && current_cpu == "mips64el" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "mips64el" && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "mipsel" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "mipsel" && ffmpeg_branding == "ChromeOS")) {
+if ((use_linux_config && current_cpu == "mips64el" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "mips64el" && ffmpeg_branding == "ChromeOS") || (use_linux_config && current_cpu == "mipsel" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "mipsel" && ffmpeg_branding == "ChromeOS") || (is_android && current_cpu == "mips64el") || (is_android && current_cpu == "mipsel")) {
   ffmpeg_c_sources += [
     "libavcodec/mips/h264chroma_init_mips.c",
     "libavcodec/mips/h264dsp_init_mips.c",
@@ -558,7 +558,7 @@ if ((use_linux_config && current_cpu == "mips64el" && ffmpeg_branding == "Chrome
   ]
 }
 
-if ((is_win && current_cpu == "arm64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm64" && ffmpeg_branding == "ChromeOS")) {
+if ((is_win && current_cpu == "arm64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm64" && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm64" && ffmpeg_branding == "ChromeOS") || (is_android && current_cpu == "arm64")) {
   ffmpeg_c_sources += [
     "libavcodec/aarch64/h264chroma_init_aarch64.c",
     "libavcodec/aarch64/h264dsp_init_aarch64.c",
@@ -572,7 +572,7 @@ if ((is_win && current_cpu == "arm64" && ffmpeg_branding == "Chrome") || (use_li
   ]
 }
 
-if (use_linux_config && current_cpu == "arm" && arm_use_neon) {
+if (use_linux_config && current_cpu == "arm" && arm_use_neon || (is_android && current_cpu == "arm" && arm_use_neon)) {
   ffmpeg_c_sources += [
     "libavcodec/arm/hpeldsp_init_neon.c",
     "libavcodec/arm/vp8dsp_init_neon.c",
@@ -622,7 +622,7 @@ if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding
   ]
 }
 
-if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "ChromeOS")) {
+if ((use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "Chrome") || (use_linux_config && current_cpu == "arm" && arm_use_neon && ffmpeg_branding == "ChromeOS") || (is_android && current_cpu == "arm" && arm_use_neon)) {
   ffmpeg_gas_sources += [
     "libavcodec/arm/h264cmc_neon.S",
     "libavcodec/arm/h264dsp_neon.S",

```

### 打包编译

编译的时候需要加入rtc_use_h264与ffmpeg_branding参数：

如下：

tools_webrtc/android/build_aar.py --build-dir out --arch "armeabi-v7a" "arm64-v8a" "x86" "x86_64" --extra-gn-args='rtc_use_h264=true ffmpeg_branding="Chrome" proprietary_codecs=true'

编译会提示没有licenses，编译报错，需要添加对应的licenses，如下：

```python
diff --git a/tools_webrtc/libs/generate_licenses.py b/tools_webrtc/libs/generate_licenses.py
index 39ba948cb8..b927df5d56 100755
--- a/tools_webrtc/libs/generate_licenses.py
+++ b/tools_webrtc/libs/generate_licenses.py
@@ -70,6 +70,12 @@ LIB_TO_LICENSES_DICT = {
     # Compile time dependencies, no license needed:
     'yasm': [],
     'ow2_asm': [],
+
+    #hwh add
+    'openh264':['third_party/openh264/src/LICENSE'],
+    'ffmpeg':['third_party/ffmpeg/LICENSE.md'],
+    'nasm':['third_party/nasm/LICENSE'],
+    # hwh end
 }
 
 # Third_party library _regex_ to licences mapping. Keys are regular expression

```

