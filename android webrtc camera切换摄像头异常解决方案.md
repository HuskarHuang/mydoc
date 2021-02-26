# android webrtc camera切换摄像头异常解决方案

camera正常切换前后置摄像头没有问题，但是快速切换的时候，会出现前后置camera出现镜像问题，并且日志报错，提示`Camera switch already in progress.`如下是点击切换按钮的代码：

```java
    //切换前后置摄像头
    controlGroupCallViewModel.getCameraOrientationLiveData().observe(this, aBoolean -> {
        if(!isCameraInit) {
            return;
        }
        Log.i(TAG, "camera = " + aBoolean);
        isBackCamera = aBoolean;
        rtcEngine().switchCamera();
        groupCallAdapter.setRenderMirror(localUid, !isBackCamera);
    });
```
可以知道，会用到rtcEngine().switchCamera()切换前后置摄像头，并且如果是前置摄像头mirror设置为true，后置摄像头设置为false，分析rtc源码，可以知道如下：

```java
public void switchCamera() {
    executor.execute(this::switchCameraInternal);
}
```
```java
private void switchCameraInternal() {
    if (videoCapturer instanceof CameraVideoCapturer) {
        if (!isVideoCallEnabled() || isError) {
            Log.e(TAG,
                    "Failed to switch camera. Video: " + isVideoCallEnabled() + ". Error : " + isError);
            return; // No video is sent or only one camera is available or error happened.
        }
        Log.d(TAG, "Switch camera");
        CameraVideoCapturer cameraVideoCapturer = (CameraVideoCapturer) videoCapturer;
        cameraVideoCapturer.switchCamera(null);
    } else {
        Log.d(TAG, "Will not switch camera, video caputurer is not a camera");
    }
}
```
由以上代码可以知道，是由于切换摄像头是在一个线程中执行的，所以当用户快速切换的时候，由于操作系统切换摄像头是需要一定时间的，所以在webrtc底层会处理不过来，所以会报`Camera switch already in progress.`

上面代码由于webrtc官方demo cameraVideoCapturer.switchCamera(null);参数为null，进去源码查看，如下：

```java
@Override
public void switchCamera(final CameraSwitchHandler switchEventsHandler) {
  Logging.d(TAG, "switchCamera");
  cameraThreadHandler.post(new Runnable() {
    @Override
    public void run() {
      switchCameraInternal(switchEventsHandler);
    }
  });
}
```

查看CameraSwitchHandler接口，如下：

```java
/**
 * Camera switch handler - one of these functions are invoked with the result of switchCamera().
 * The callback may be called on an arbitrary thread.
 */
public interface CameraSwitchHandler {
  // Invoked on success. |isFrontCamera| is true if the new camera is front facing.
  void onCameraSwitchDone(boolean isFrontCamera);

  // Invoked on failure, e.g. camera is stopped or only one camera available.
  void onCameraSwitchError(String errorDescription);
}
```

所以，可以在onCameraSwitchDone，即切换成功后，做镜像操作，

最后通过给应该层switchCamera一个回调接口，如下：

```java
/**
 * 切换前后置摄像头，sdk默认打开的是前置摄像头
 * @param cameraSwitchHandler 切换摄像头后的回调
 * @return
 */
public abstract int switchCamera(CameraVideoCapturer.CameraSwitchHandler cameraSwitchHandler);
```

最后改动应用层源码，如下：

```java
rtcEngine().switchCamera(new CameraVideoCapturer.CameraSwitchHandler() {
    @Override
    public void onCameraSwitchDone(boolean isFrontCamera) {
        groupCallAdapter.setRenderMirror(localUid, isFrontCamera);
    }

    @Override
    public void onCameraSwitchError(String errorDescription) {
        Log.e(TAG, "error = " + errorDescription);
    }
});
```

即如果用户快速切换，能够返回切换是否成功，并且这次成功的是前置摄像头还是后置摄像头，并且如果是发生错误了，就不去做镜像处理，这样就不导致快速切换会导致镜像的问题



