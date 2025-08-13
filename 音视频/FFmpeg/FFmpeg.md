官方文档：https://www.ffmpeg.org/documentation.html

# 简介
FFmpeg框架的基本组成包含AVFormat、AVCodec、AVFilter、AVDevice、AVUtil等模块库。
* AVFormat 实现了绝大多数媒体封装和解封装格式，如 mp4、flv等文件封装格式，rtp、rtsp、hls等网络协议封装格式。还可以增加自己定制的封装格式。
* AVCodec 实现了绝大多数的编解码格式，如果希望增加自己的编码格式或者使用硬件编解码，则需要在AVCodec中增加相应的编解码模块。
* AVFilter 提供了一个通用的音频、视频、字幕等滤镜处理框架，可以方便的对视频进行滤镜处理，如模糊、锐化、去色、缩放、旋转、裁剪等。
* AVUtil 
* swscale 提供高级别的图像转换API，例如图像缩放（比如把1080p转换为720p）、像素格式转换（比如把YUV420P转换为YUVY、RGB）
* swresample 提供高级别的音频重采样API，例如音频格式转换、音频采样、音频通道布局转换等。


ffmpeg 是FFmpeg源代码编译生成的可执行程序，可以作为命令行工具使用。

mp4转avi：使用AVFormat解封装、AVCodec解码、AVCodec编码、AVFormat封装


ffplay是FFmpeg源代码编译生成的另一个可执行程序，可以用于音视频播放图像、音频波形

ffprobe也是FFmpeg源代码编译生成的一个可执行程序，它是一个媒体文件信息查看工具，可以查看视频、音频、图片、字幕等媒体文件的信息。

FFmpeg所做的只是提供一套基础的框架，所有的编码格式、文件封装格式与流媒体协议均可以作为FFmpeg的一个模块挂载在FFmpeg框架中。

FFmpeg的源码中提供了 `configure` 可执行文件，可以用于定制化，比如只想要h264 aac mp4，这样可以减少包体积，通过 configure 完成配置后，执行make即可编译出FFmpeg。

`configure`还可以用于查看FFmpeg支持的所有编解码器、封装格式和通信协议，例如 `./configure --list-encoders`可以查看FFmpeg支持的所有编码器。

# 使用基础
FFmpeg中常用的工具是ffmpeg、ffprobe、ffplay，分别用于编解码、内容分析、播放器。

## ffmpeg命令规则
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...

## ffprobe常用命令
ffprobe --help 可以查看使用方式

## ffplay
ffplay不仅是播放器，同时也是测试ffmpeg的codec引擎、format引擎以及filter引擎的工具，并且还可以可视化地分析媒体参数。使用ffplay --help查看使用方式。

# FFmpeg转封装
ffmpeg -h demuxer=mp4 可以查看有哪些用于mp4文件的Demuxer

ffmpeg -i input.mp4 -vn -acodec copy output.aac 从mp4文件中分离出音频流，并转码为aac格式
ffmpeg -i input.mp4 -vcodec copy -an output.h264 从mp4文件中分离出视频流，并转码为h264格式

# FFmpeg转码
h264是当前使用最普遍的视频编码格式，通过`ffmpeg -h encoder=libx264`可以查看它支持的像素格式，比如yuv420p、nv12等
https://www.ffmpeg.org/ffmpeg-codecs.html#libx264_002c-libx264rgb 可以查看libx264的参数

## FFmpeg硬编解码
FFmpeg也可以利用GPU编解码

## mp3
ffmpeg -i INPUT -c:a libmp3lame OUTPUT.mp3 将音频编码为mp3文件

## aac
ffmpeg -i input -c:a aac output.aac

# FFmpeg音视频流媒体
RTMP直播
https://www.ffmpeg.org/ffmpeg-all.html#rtmp

ffmpeg -rtmp_app live -rtmp_playpath class -i rtmp://xxx.com -c copy -f flv output.flv

ffmpeg -i rtmp://xxx.com/live/class -c copy -f flv output.flv

可以从RTMP服务器拉取直播流，保存为flv文件

还有http协议

还支持tcp、udp这些低层级的协议

# FFmpeg滤镜
AVFilter组件常用于多媒体的处理和编辑

https://www.ffmpeg.org/ffmpeg-filters.html 滤镜相关的所有操作

ffmpeg -i input.mp4 -i logo.png -filter_complex "[1:v]scale=176:144[logo];[0:v][logo]overlay=x=0:y=0" output.mp4
将logo.png的图像缩放为176*144的分辨率，然后定义一个临时名称logo，并将缩放后的logo叠加到input.mp4的视频流[0:v]的左上角。

滤镜可以操作视频、图像、音频、字幕等

# FFmpeg采集设备
ffmpeg -hide_banner -devices 可以查看当前系统支持的输入和输出的设备

ffmpeg -framerate 30 -f fbdev -i /dev/fb0 output.mp4 Linix将获取终端中的图像，而不是图形界面的图像可以通过这种方式录制Linux终端中的操作

还有其他设备

设备相关命令文档 https://www.ffmpeg.org/ffmpeg-devices.html

# API使用
前面主要介绍命令行的使用，现在介绍API的使用方式

FFmpeg 命令行只是一个使用 FFmpeg API 的应用程序，我们也可以直接使用API编程实现功能

* AVFormatContext：容器格式上下文，或者叫封装格式上下文。可以理解为 MP4 或者 FLV 等容器。
* AVPacket：数据包（已编码压缩的数据），这里面的数据通常是一帧视频的数据，或者一帧音频的数据。
* AVCodecContext：这个结构体可以是 编码器 的上下文，也可以是 解码器 的上下文，两者使用的是同一种数据结构。
* AVCodec
* AVFrame
* AVCodecParameters
* AVDictionary
* AVStream


https://www.ffmpeg.org/documentation.html 中的`API Documentation`部分就是API文档

比如7.0的版本 https://www.ffmpeg.org/doxygen/7.0/index.html


一些`libavformat`（用于封装/解封装）内的API：
https://www.ffmpeg.org/doxygen/7.0/group__libavf.html
```
avformat_open_input

avformat_alloc_output_context2

avformat_new_stream

avcodec_parameters_copy

avformat_write_header

av_read_frame

av_write_trailer
```

官方代码示例：https://www.ffmpeg.org/doxygen/7.0/examples.html


`libavcodec`（用于编解码）的API

https://www.ffmpeg.org/doxygen/7.0/group__libavc.html


`libavfilter`

https://www.ffmpeg.org/doxygen/7.0/group__lavfi.html

# 编译、裁剪
FFmpeg的编译和裁剪都是通过configure来完成的，configure是FFmpeg的配置脚本文件，它通过解析配置文件来决定编译哪些模块，比如编译哪些编码器、解码器、封装格式、通信协议等。通过执行`./configure`相关的配置命令，会生成config.h和config_components.h两个文件，其他文件夹内的文件也会相应修改，比如 libavcodec 目录内也会生成 codec_list.c 等配置文件。


./configure --help 可以查看所有命令


FFmepg编译的基本流程如下：

1. 选择编译工具
2. 配置交叉编译环境
3. 配置编译参数（比如去掉一些不需要的功能）
4. 启动编译

Android ndk编译的情况：
```
编译工具链目录：
toolchains/llvm/prebuilt/darwin-x86_64/bin

交叉编译环境目录：
toolchains/llvm/prebuilt/darwin-x86_64/sysroot
// usr/include usr/lib 这两个路径，相关的头文件和库文件
```

因为 configure 涉及的配置很多，所以一般会使用脚本文件来完成配置，比如：
```
#!/bin/bash

API=21
NDK=$ANDROID_NDK_HOME
TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/linux-x86_64
SYSROOT=$TOOLCHAIN/sysroot

ARCHS=("arm" "arm64" "x86" "x86_64")
ABIS=("armeabi-v7a" "arm64-v8a" "x86" "x86_64")
TARGET_CPU=("armv7-a" "armv8-a" "i686" "x86_64")

for i in "${!ARCHS[@]}"; do
  ARCH=${ARCHS[$i]}
  ABI=${ABIS[$i]}
  TARGET=${TARGET_CPU[$i]}

  OUTPUT=android/$ABI
  mkdir -p $OUTPUT

  ./configure \
    --prefix=$OUTPUT \
    --enable-shared \
    --disable-static \
    --disable-doc \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-symver \
    --disable-programs \
    --target-os=android \
    --arch=$ARCH \
    --cpu=$TARGET \
    --cc=$TOOLCHAIN/bin/${ARCH}-linux-android$API-clang \
    --cxx=$TOOLCHAIN/bin/${ARCH}-linux-android$API-clang++ \
    --strip=$TOOLCHAIN/bin/llvm-strip \
    --sysroot=$SYSROOT \
    --extra-cflags="-march=$TARGET -Os -fPIC" \
    --extra-ldflags="-march=$TARGET"

  make clean
  make install
done
```
总的来说，就是使用 configure 脚本完成配置，然后使用 make 命令来编译和安装。