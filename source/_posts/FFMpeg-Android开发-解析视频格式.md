---
title: FFMpeg-Android开发-解析视频格式
date: 2020-08-22 09:07:51
tags:
---

在正式使用FFMpeg完成视频的播放之前，我想先写一篇文章，简单介绍一下如何使用FFMpeg去获取一个视频的基本信息。通过这篇文章，来简单介绍一下FFmpeg中的相关概念，以及视频的一些基本概念。

## 视频基本概念

当我们提及视频格式的时候，实际上对应的是视频的container(又或者称为format)这个概念，这个概念往往对应mpeg4，mkv, webm等。一个container可以是由多个编码器压缩后的媒体的集合，例如，一个mp4，可以包含视频，音频，字幕等不同媒体

编码器codec，则是按照某种编解码标准下的具体实现，这一层对应的概念往往是av1, h264, vp9等

因此，一个视频文件，从读取IO到播放大致可以分为以下几步：

1. 解析container
2. 根据container，获得我们关心的媒体数据（例如播放视频，那我们只关心视频媒体）
3. 根据媒体信息获得对应的解码器
4. 将对应的数据送给解码器
5. 解码器解码，输出帧
6. 将帧渲染到目标区域

## FFmpeg中对上述概念的定义

那么在FFmpeg中，是如何定义上述概念的呢？

对于container，FFMpeg使用`AVFormatContext`
对于具体的媒体数据，通过遍历`AVFormatContext->streams`，我们能通过一个`AVStream`对象表示

`AVStream`中的压缩分片用`AVPacket`表示，通过解码器解码后我们就能得到帧数据，用`AVFrame`表示

这篇文章就先简单介绍下1，2，3步的实现

## 使用FFMpeg解析container，获取基本信息


代码如下

```c++

AVFormatContext *avFormatContext = nullptr;
LOGI("video_config_create, open file uri: %s", uri);
if (avformat_open_input(&avFormatContext, uri, nullptr, nullptr)) { // open IO, 如果成功, avFormatContext将有值
    return nullptr;
}


if (avformat_find_stream_info(avFormatContext, nullptr) < 0) {
    avformat_free_context(avFormatContext);
    return nullptr;
};

for (int pos = 0; pos < avFormatContext->nb_streams; pos++) {
    // Getting the name of a codec of the very first video stream
    AVCodecParameters *parameters = avFormatContext->streams[pos]->codecpar;
    if (parameters->codec_type == AVMEDIA_TYPE_VIDEO) {
        videoConfig->parameters = parameters;
        videoConfig->avVideoCodec = avcodec_find_decoder(parameters->codec_id);
        videoConfig->videoStreamIndex = pos;
        break;
    }
}
```

代码的基本逻辑，和上述介绍的步骤，是一致的；其中，视频的文件格式信息，显然就在`AVFormatContext`中，视频的宽高信息，就在`AVCodecParameters`中，这个对象顾名思义，是跟随codec的；而解码器的相关信息，就在`avcodec_find_decoder`的返回值中


本文对应的源代码请参考[https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.0](https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.0)