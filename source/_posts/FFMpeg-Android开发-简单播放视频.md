---
title: FFMpeg-Android开发-简单播放视频
date: 2020-08-23 10:25:15
tags:
---

本文部分对应的源代码可参考[https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.1](https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.1)

## 具体实现

结合上一篇文章`FFMpeg-Android开发-解析视频格式`的内容，我们知道视频解码大致分为如下几个步骤

1. 解析 container
2. 根据 container，获得我们关心的媒体数据（例如播放视频，那我们只关心视频媒体）
3. 根据媒体信息获得对应的解码器
4. 将对应的数据送给解码器
5. 解码器解码，输出帧
6. 将帧渲染到目标区域

那么这次我们就来尝试将视频解码并渲染到设备屏幕上，首先看我们 1-3 步的代码

```c++

// Record result
int result;
// R1 Java String -> C String
const char *path = env->GetStringUTFChars(path_, 0);
// Register FFmpeg components
av_register_all();
// R2 initializes the AVFormatContext context
AVFormatContext *format_context = avformat_alloc_context();
// Open Video File
result = avformat_open_input(&format_context, path, NULL, NULL);
if (result < 0) {
    LOGE("Player Error : Can not open video file");
    return;
}
// Finding Stream Information of Video Files
result = avformat_find_stream_info(format_context, NULL);
if (result < 0) {
    LOGE("Player Error : Can not find video file stream info");
    return;
}
// Find Video Encoder
int video_stream_index = -1;
for (int i = 0; i < format_context->nb_streams; i++) {
    // Matching Video Stream
    if (format_context->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
        video_stream_index = i;
    }
}
// No video stream found
if (video_stream_index == -1) {
    LOGE("Player Error : Can not find video stream");
    return;
}
// Initialization of Video Encoder Context
AVCodecContext *video_codec_context = avcodec_alloc_context3(NULL);
avcodec_parameters_to_context(video_codec_context, format_context->streams[video_stream_index]->codecpar);
// Initialization of Video Encoder
AVCodec *video_codec = avcodec_find_decoder(video_codec_context->codec_id);
if (video_codec == NULL) {
    LOGE("Player Error : Can not find video codec");
    return;
}
// R3 Opens Video Decoder
result  = avcodec_open2(video_codec_context, video_codec, NULL);
if (result < 0) {
    LOGE("Player Error : Can not find video stream");
    return;
}
// Getting the Width and Height of Video
int videoWidth = video_codec_context->width;
int videoHeight = video_codec_context->height;
```

其中 video_codec 代表解码器对象，其他对象在上一篇文章和代码注释中有相关解释。

接下来的步骤就是解码和上屏，这里我们先看下我们的上屏是如何实现的；这里为了方便，我直接使用 SurfaceView 构建我们的上层 View

```kotlin
public fun onCreate(savedInstanceState: Bundle?) {
    // ...
    surfaceHolder = surfaceView.holder
    surfaceHolder!!.setFormat(PixelFormat.RGBA_8888)
}

private fun playVideo(path: String) {
    Thread {
        mPlayer.playVideo(path, surfaceHolder!!.surface)
    }.start()

}
```

```c++
// R4 Initializes Native Window s for Playing Videos
ANativeWindow *native_window = ANativeWindow_fromSurface(env, surface); // surface对应java层的surface对象
if (native_window == NULL) {
    LOGE("Player Error : Can not create native window");
    return;
}
// Limit the number of pixels in the buffer by setting the width, not the physical display size of the screen.
// If the buffer does not match the display size of the physical screen, the actual display may be stretched or compressed images.
result = ANativeWindow_setBuffersGeometry(native_window, videoWidth, videoHeight,WINDOW_FORMAT_RGBA_8888);
if (result < 0){
    LOGE("Player Error : Can not set native window buffer");
    ANativeWindow_release(native_window);
    return;
}
// Define drawing buffer
ANativeWindow_Buffer window_buffer;
// There are three declarative data containers
// Data container Packet encoding data before R5 decoding
AVPacket *packet = av_packet_alloc();
av_init_packet(packet);
// Frame Pixel Data of Data Container After R6 Decoding Can't Play Pixel Data Directly and Need Conversion
AVFrame *frame = av_frame_alloc();
// R7 converted data container where the data can be used for playback
AVFrame *rgba_frame = av_frame_alloc();
// Data format conversion preparation
// Output Buffer
int buffer_size = av_image_get_buffer_size(AV_PIX_FMT_RGBA, videoWidth, videoHeight, 1);
// R8 Application for Buffer Memory
uint8_t *out_buffer = (uint8_t *) av_malloc(buffer_size * sizeof(uint8_t));
LOGI("outBuffer size: %d, videoWidth: %d, videoHeight: %d, pix_fmt: %d", buffer_size * sizeof(uint8_t), videoWidth, videoHeight, video_codec_context->pix_fmt);
av_image_fill_arrays(rgba_frame->data, rgba_frame->linesize, out_buffer, AV_PIX_FMT_RGBA, videoWidth, videoHeight, 1);
// R9 Data Format Conversion Context
struct SwsContext *data_convert_context = sws_getContext(
        videoWidth, videoHeight, video_codec_context->pix_fmt,
        videoWidth, videoHeight, AV_PIX_FMT_RGBA,
        SWS_BICUBIC, NULL, NULL, NULL);
// Start reading frames

```

这里 ANative 相关代码都是 Android NDK 中的 api，可参考文档理解；这里主要看一下这里的 AVPacket 和 AVFrame 的使用

AVPacket 在上一篇文章中有过介绍，它在 FFMpeg 中用来表示未解码时的数据；而 AVFrame 则表示了解码后的帧数据。

但这里有一个小问题是我们解码后的图像格式可能是和我们的 surface 的渲染格式不同，这里我们的 surface 渲染格式是`RGBA8888`，而视频的图像格式不一定为这个格式。为了实现转换，我们就需要`SwsContext`对象来帮助我们实现图像格式的转换。

看完上面后，终于来到了我们的解码 runloop

```c++
while (av_read_frame(format_context, packet) >= 0) {
    // Matching Video Stream
    if (packet->stream_index == video_stream_index) {
        // Decode video
        result = avcodec_send_packet(video_codec_context, packet);
        if (result < 0 && result != AVERROR(EAGAIN) && result != AVERROR_EOF) {
            LOGE("Player Error : codec step 1 fail");
            return;
        }
        result = avcodec_receive_frame(video_codec_context, frame);
        if (result < 0 && result != AVERROR_EOF) {
            LOGE("Player Error : codec step 2 fail");
            return;
        }
        LOGI("Frame %c (%d, size=%d) pts %d dts %d key_frame %d [codec_picture_number %d, display_picture_number %d]",
                av_get_picture_type_char(frame->pict_type), video_codec_context->frame_number,
                frame->pkt_size,
                frame->pts,
                frame->pkt_dts,
                frame->key_frame, frame->coded_picture_number, frame->display_picture_number);
        // Data Format Conversion
        result = sws_scale(
                data_convert_context,
                frame->data, frame->linesize,
                0, videoHeight,
                rgba_frame->data, rgba_frame->linesize);
        // play
        result = ANativeWindow_lock(native_window, &window_buffer, NULL);
        if (result < 0) {
            LOGE("Player Error : Can not lock native window");
        } else {
            // Draw the image onto the interface
            // Note: The pixel lengths of rgba_frame row and window_buffer row may not be the same here.
            // Need to convert well or maybe screen
            uint8_t *bits = (uint8_t *) window_buffer.bits;
            for (int h = 0; h < videoHeight; h++) {
                memcpy(bits + h * window_buffer.stride * 4,
                        out_buffer + h * rgba_frame->linesize[0],
                        rgba_frame->linesize[0]);
            }
            ANativeWindow_unlockAndPost(native_window);
        }
    }
    // Release package references
    av_packet_unref(packet);
}

```

整个解码循环也是大致分为如下几个步骤

1. 通过 AVPacket 读取一块数据
2. 将 AVPacket 送给解码器使用
3. 通过 AVFrame 得到 2 中的解码后数据
4. 对 AVFrame 的图像格式进行转换
5. 将 4 中的结果上屏

其他的注释在代码中有说明

最后在 run-loop 外完成资源回收

```c++
// Release R9
sws_freeContext(data_convert_context);
// Release R8
av_free(out_buffer);
// Release R7
av_frame_free(&rgba_frame);
// Release R6
av_frame_free(&frame);
// Release R5
av_packet_free(&packet);
// Release R4
ANativeWindow_release(native_window);
// Close R3
avcodec_close(video_codec_context);
// Release R2
avformat_close_input(&format_context);
// Release R1
env->ReleaseStringUTFChars(path_, path);

```

此时，我们就完成了整个视频的播放，不过这个播放器显然是处在不可用的状态，主要有下面两个问题

1. 没有声音
2. 视频的播放是按照解码器的解码速度播放的；只要解码器足够快，视频播放就有多快，这个显然是不符合播放器播放视频的预期的

问题 2 本质就是我们常说的音视频同步问题；我们后面会简单介绍音频的解码播放后，通过引入 pts, dts 等概念后，尝试解决问题 2
