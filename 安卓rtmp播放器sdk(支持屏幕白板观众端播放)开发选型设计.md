# 安卓rtmp播放器sdk(支持屏幕白板观众端播放)开发选型设计

由于屏幕白板使用到的播放器的网络协议是rtmp，安卓自带的播放器无法解析此协议，故需要找第三方播放器来解决网络协议解封装的问题，目前安卓上最受欢迎的是bilibili公司开源的ijkplayer播放器，而且其底层使用的是基于ffmpeg里面的ffplay里面做的修改处理，支持硬解与软件，并且也支持ios平台，故选择此播放器内核作为我们项目屏幕白板的播放端使用

下面我们通过下载ijkplayer源码，并且在ubuntu系统进行编译，生成ffmpeg-arm64、ffmpeg-armv7a、ffmpeg-x86_64、ffmpeg-x86平台so库，即生成各个平台so库，arm平台可以在手机使用，x86可以在模拟器使用，这样后续如果有什么要定制化相关的东西，由于所有so库和源码都自己生成，方便后续定制化开发

首先要配置好android ndk，刚开始使用的太高版本的ndk，无法编译通过，后面通过尝试ndk-r14b目前是可以编译通过的

下面是nkd-r14b的下载链接

https://dl.google.com/android/repository/android-ndk-r14b-linux-x86_64.zip

下载解压后，设置ANDROID_NDK环境变量，后续编译会用到，如下

```shell
export ANDROID_NDK=/home/hwh/android-ndk-r14b-linux-x86_64/android-ndk-r14b
```

克隆代码，并且cd到代码目录

```shell
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
cd ijkplayer-android
```

切换到最新稳定Tags

```shell
git checkout -B latest k0.8.8
```

下载ffmpeg与其他相关源码到android/contrib/下面，并且初始相关环境变量，cd到android/contrib

```shell
./init-android.sh
cd android/contrib
```

clean一下，输入指令

```shell
./compile-ffmpeg.sh clean
```

编译ffmpeg库

```shell
./compile-ffmpeg.sh all
```

这样就在如下目录生成各个平台的libijkffmpeg.so库

```
android/contrib/build/ffmpeg-arm64/output
android/contrib/build/ffmpeg-armv7a/output
android/contrib/build/ffmpeg-x86/output
android/contrib/build/ffmpeg-x86_64/output
```

cd到上一层，执行compile-ijk.sh命令

```shell
cd ../
./compile-ijk.sh all
```

这样就在下面的目录生成libijkplayer.so与libijksdl.so

```shell
ijkplayer/ijkplayer-arm64/src/main/libs/arm64-v8a
ijkplayer/ijkplayer-armv7a/src/main/libs/armeabi-v7a
ijkplayer/ijkplayer-x86_64/src/main/libs/x86_64
ijkplayer/ijkplayer-x86/src/main/libs/x86
```

将java层module(ijkplayer/ijkplayer-java目录)与这三个库(libijkffmpeg.so、libijkplayer.so、libijksdl.so)相关平台复制demo上，目录结构如下

```
ijkplayer-java
 src
  main
   java  -> 提供给app上层使用的java api接口sdk
   jniLibs  -> 下面是复制过来的库相关目录
    arm64-v8a
     libijkffmpeg.so
     libijkplayer.so
     libijksdl.so
    arm64-v8a
     libijkffmpeg.so
     libijkplayer.so
     libijksdl.so
    armeabi-v7a
     libijkffmpeg.so
     libijkplayer.so
     libijksdl.so
    x86_64
     libijkffmpeg.so
     libijkplayer.so
     libijksdl.so
    x86
     libijkffmpeg.so
     libijkplayer.so
     libijksdl.so
```

目前ijkplayer播放器上层api接口已经有了，接下来是找一个功能强大的皮肤并且可以对ijkplay上层api进行封装，能够兼容不同的皮肤支持且给上层用户使用，能够让用户很方便的在不同的播放器内核进行切换，目前找到一个开源的项目[JZVideo](https://github.com/Jzvd/JZVideo)

在原来的JZVideo进行剥离处理，将ui库放到一个单独的module，并且ui库再引用ijkplayer-java module，这样用户app层只需要导入ui皮肤控件，如下：

```xml
<cn.jzvd.JzvdStd
    android:id="@+id/videoPlayer"
    android:layout_width="0dp"
    android:layout_height="200dp"
    app:layout_constraintEnd_toEndOf="parent"
    app:layout_constraintStart_toStartOf="parent"
    app:layout_constraintTop_toTopOf="parent" />
```

在代码里面只需要调用

```java
private static final String URL = "rtmp://58.200.131.2:1935/livetv/gdtv";

JzvdStd mJzvdStd = findViewById(R.id.videoPlayer);//找到播放器ui控件
JZDataSource jzDataSource = new JZDataSource(URL, "标题自定义设置");//设置播放url与标题

mJzvdStd.setUp(jzDataSource, JzvdStd.SCREEN_NORMAL, JZMediaIjk.class);//指定用ijkplayer内核播放
Glide.with(this).load(PIC_URL).into(mJzvdStd.posterImageView);//还没播放时的封面显示

```

JZMediaIjk实现JZMediaInterface的上层播放器接口，使用到的是ijkplayer-java module接口，如下：

```java
public abstract class JZMediaInterface implements TextureView.SurfaceTextureListener {

    public static SurfaceTexture SAVED_SURFACE;
    public HandlerThread mMediaHandlerThread;
    public Handler mMediaHandler;
    public Handler handler;
    public Jzvd jzvd;


    public JZMediaInterface(Jzvd jzvd) {
        this.jzvd = jzvd;
    }

    public abstract void start();

    public abstract void prepare();

    public abstract void pause();

    public abstract boolean isPlaying();

    public abstract void seekTo(long time);

    public abstract void release();

    public abstract long getCurrentPosition();

    public abstract long getDuration();

    public abstract void setVolume(float leftVolume, float rightVolume);

    public abstract void setSpeed(float speed);

    public abstract void setSurface(Surface surface);
}
```

如果要支持exo-player或者阿里播放内核，只需要实现JZMediaInterface的接口即可，使用时指定class即可，如下：

```java
mJzvdStd.setUp(jzDataSource, JzvdStd.SCREEN_NORMAL, JZMediaAliyun.class);//指定用阿里内核播放
mJzvdStd.setUp(jzDataSource, JzvdStd.SCREEN_NORMAL, JZMediaExo.class);//指定用exo内核播放
```