# android webrtc so库bug c++层崩溃分析与解决

## c++或c层闪退，java层无法看出具体闪退出现在哪里

### 问题描述 -> 群聊:用户A/B/C正在通话,用户C点击挂断同时,用户D点击视频通话;此时用户D大概率无法加入房间,表现为app界面卡死

抓到如下log日志：

```
2020-11-13 14:29:36.591 10446-11703/? E/CrashReport: #++++++++++Record By Bugly++++++++++#
2020-11-13 14:29:36.591 10446-11703/? E/CrashReport: # You can use Bugly(http:\\bugly.qq.com) to get more Crash Detail!
2020-11-13 14:29:36.591 10446-11703/? E/CrashReport: # PKG NAME: com.mongo.im.mimsdk
2020-11-13 14:29:36.591 10446-11703/? E/CrashReport: # APP VER: 1.0
2020-11-13 14:29:36.593 10446-11703/? E/CrashReport: # LAUNCH TIME: 2020-11-13 14:28:41
2020-11-13 14:29:36.593 10446-11703/? E/CrashReport: # CRASH TYPE: NATIVE_CRASH
2020-11-13 14:29:36.593 10446-11703/? E/CrashReport: # CRASH TIME: 2020-11-13 14:29:36
2020-11-13 14:29:36.594 10446-11703/? E/CrashReport: # CRASH PROCESS: com.mongo.im.mimsdk
2020-11-13 14:29:36.594 10446-11703/? E/CrashReport: # CRASH THREAD: pool-11-thread-1(777)
2020-11-13 14:29:36.594 10446-11703/? E/CrashReport: # REPORT ID: d547436d-a829-4f8a-9d38-bf30bbd760bc
2020-11-13 14:29:36.595 10446-11703/? E/CrashReport: # CRASH DEVICE: PDBM00 UNROOT
2020-11-13 14:29:36.595 10446-11703/? E/CrashReport: # RUNTIME AVAIL RAM:2015485952 ROM:105446072320 SD:105236357120
2020-11-13 14:29:36.595 10446-11703/? E/CrashReport: # RUNTIME TOTAL RAM:3916988416 ROM:114577584128 SD:114367868928
2020-11-13 14:29:36.595 10446-11703/? E/CrashReport: # EXCEPTION FIRED BY KERNEL UNKNOWN
2020-11-13 14:29:36.596 10446-11703/? E/CrashReport: # CRASH STACK: 
2020-11-13 14:29:36.596 10446-11703/? E/CrashReport: SIGSEGV(SEGV_MAPERR)
    0x8
    #00    pc 002faf38    /data/app/com.mongo.im.mimsdk-VjU3kXxffvGTAP5Iu8ZoYw==/lib/arm/libjingle_peerconnection_so.so (Java_org_webrtc_PeerConnection_nativeGetLocalDescription+13) [armeabi::5a75b251aea1b4db790c14fb65bf78b3]
    #01    pc 0040ac79    /system/lib/libart.so [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #02    pc 00406775    /system/lib/libart.so (ExecuteMterpImpl+18933) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #03    pc 003dfb05    /system/lib/libart.so (art_quick_invoke_stub+224) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #04    pc 00095a15    /system/lib/libart.so (_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc+136) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #05    pc 001dbad1    /system/lib/libart.so (_ZN3art11interpreter34ArtInterpreterToCompiledCodeBridgeEPNS_6ThreadEPNS_9ArtMethodEPNS_11ShadowFrameEtPNS_6JValueE+236) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #06    pc 001d65bf    /system/lib/libart.so (_ZN3art11interpreter6DoCallILb0ELb0EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE+814) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #07    pc 003db5e9    /system/lib/libart.so (MterpInvokeDirect+196) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #08    pc 003f9614    /system/lib/libart.so (deflate+4292) [armeabi-v8::974c4036f10d001a4533f21d640c7517]
    #09    pc c330828c    <unknown>
    java:
    org.webrtc.PeerConnection.getLocalDescription(PeerConnection.java:874)
    com.zx.zxlivs.apprtc.PeerConnectionClient$SDPObserver.lambda$onSetSuccess$1$PeerConnectionClient$SDPObserver(PeerConnectionClient.java:1557)
    com.zx.zxlivs.apprtc.-$$Lambda$PeerConnectionClient$SDPObserver$SE2Nzmhlj1inAtBPknuc6im0R0k.run(Unknown Source:2)
    java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    java.lang.Thread.run(Thread.java:764)
2020-11-13 14:29:36.596 10446-11703/? E/CrashReport: #++++++++++++++++++++++++++++++++++++++++++#
```

从以上可以看出，java层出现异常如下：

```java
if (peerConnection == null || isError) {
  return;
}
if (type) {
  if (peerConnection.getRemoteDescription() == null) {
    events.onLocalDescription(handleId, localSdp);
  } else {
    Log.d(TAG, "Remote SDP set succesfully");
    drainCandidates(handleId);
  }
} else {//收到服务发过来offer 这时为 false
  if (peerConnection.getLocalDescription() != null) {  ->出错的地方
    Log.d(TAG, "Local SDP set succesfully");
    events.onLocalDescription(handleId, localSdp);
    drainCandidates(handleId);
  } else {
    Log.d(TAG, "Remote SDP set succesfully");
  }
}
```

peerConnection.getLocalDescription()出现异常了，无法直接看出来是哪里出错了，上面log日志可以知道libjingle_peerconnection_so.so出现异常，如下：

```
    #00    pc 002faf38    /data/app/com.mongo.im.mimsdk-VjU3kXxffvGTAP5Iu8ZoYw==/lib/arm/libjingle_peerconnection_so.so (Java_org_webrtc_PeerConnection_nativeGetLocalDescription+13) [armeabi::5a75b251aea1b4db790c14fb65bf78b3]
```

出错的pc指针为002faf38，进入到ndk对应arm-linux-androideabi-addr2line文件夹目录：

cd webrtc_android/src/third_party/android_ndk/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin

执行命令，并将002faf38地址作为参数传入：

./arm-linux-androideabi-addr2line -e /home/hwh/webrtc/webrtc_android/src/out/armeabi-v7a/lib.unstripped/libjingle_peerconnection_so.so 002faf38

会打印具体出错的地方，命令行控制台会输出如下：

./../../sdk/android/src/jni/pc/peer_connection.cc:491

由此可以知道出错是在peer_connection.cc 491行

找到代码如下：

```c++
487 static ScopedJavaLocalRef<jobject> JNI_PeerConnection_GetLocalDescription(
488    JNIEnv* jni,
489    const JavaParamRef<jobject>& j_pc) {
490  const SessionDescriptionInterface* sdp =
491      ExtractNativePC(jni, j_pc)->local_description();
492  return sdp ? NativeToJavaSessionDescription(jni, sdp) : nullptr;
493 }
```

分析定位，发现是ExtractNativePC(jni, j_pc)为空，做一下非空判断，问题解决