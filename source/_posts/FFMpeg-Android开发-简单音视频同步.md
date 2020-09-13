---
title: FFMpeg-Android开发-简单音视频同步
date: 2020-09-13 10:39:03
tags:
---

本文代码可参考代码可以参考[https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v2.1](https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v2.1)

我们之前实现了视频和音频的播放，但其中最大的问题是我们的音频和视频之间的播放速度没有同步，视频按照解码的速度，以最快速度进行了上屏，那么很有可能会出现视频播放完后音频还在播放的情况。这次我们就来尝试解决这个问题，正式解决问题前，我们先对一些基本概念做出介绍

## FFMpeg 中的 dts 和 pts

FFmpeg 里有两种时间戳：DTS（Decoding Time Stamp）和 PTS（Presentation Time Stamp）。前者是解码的时间，后者是显示的时间。要仔细理解这两个概念，需要先了解 FFmpeg 中的 packet 和 frame 的概念。

FFmpeg 中用 AVPacket 结构体来描述解码前或编码后的压缩包，用 AVFrame 结构体来描述解码后或编码前的信号帧。 对于视频来说，AVFrame 就是视频的一帧图像。这帧图像什么时候显示给用户，就取决于它的 PTS。DTS 是 AVPacket 里的一个成员，表示这个压缩包应该什么时候被解码。 如果视频里各帧的编码是按输入顺序（也就是显示顺序）依次进行的，那么解码和显示时间应该是一致的。可事实上，在大多数编解码标准（如 H.264 或 HEVC）中，编码顺序和输入顺序并不一致。 于是才会需要 PTS 和 DTS 这两种不同的时间戳。

具体到代码中为

```c++
AVPacket *packet = av_packet_alloc();
av_init_packet(packet);


// packet->dts == AV_NOPTS_VALUE
while (av_read_frame(format_context, packet) >= 0) {
    // do decoding ...
    // packet->dts != AV_NOPTS_VALUE here

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
}

```

那么为什么要区分 DTS 和 PTS 呢？这里就涉及到编解码标准中编码顺序和输入顺序不一致的问题，这里以 H.264 规范为例举一个例子

假设有一个解码帧序列为`IBP`，当解码器对其解码时，解码第一帧肯定是序列中的第一帧 I, 但我们思考一下第二帧，已知 B 帧的解码需要参考前一个 I 帧或 P 帧以及后面一个 P 帧，那么解码的第二帧显然不能是序列中的第二帧 B 帧，而是第三帧 P 帧，因此我们得到了下面的一个 PTS 和 DTS 关系

    PTS: 1 2 3
    DTS: 1 3 2

这里我们就会发现，DTS 和 PTS 是不同的

DTS 主要用于视频的解码,在解码阶段使用.PTS 主要用于视频的同步和输出.在 display 的时候使用.在没有 B frame 的情况下.DTS 和 PTS 的输出顺序是一样的.

## Timebase

在 FFMpeg 中，同时还引入了 timebase 这个概念。timebase 用来度量时间尺度，假设 timebase={1, 25}, 那么意味着时间尺度就是 1/25 秒，假设 pts=20，那么在 timebase={1, 25}的情况下。这一帧的时间为`(20 * 1/25)s`

FFMpeg 中，不同的数据状态对应的 timebase 也不一致。例如，非压缩时的数据(即 YUV 或者其它)，在 ffmpeg 中对应的结构体为 AVFrame,它的 timebase 为 AVCodecContext 的 time_base ,AVRational{1,25}。
压缩后的数据(对应的结构体为 AVPacket)对应的 timebase 为 AVStream 的 time_base，AVRational{1,90000}。
因为数据状态不同，timebase 不一样，所以我们必须转换，在 1/25 时间刻度下佔 10 格，在 1/90000 下是佔多少格。这就是 pts 的转换。

## 利用 pts 实现音视频同步

当我们得到 timebase 和 pts 数据后，我们就可以通过时间换算来同步视频和音频的播放，考虑到音频的播放速度固定，最简单的做法就是将视频的播放向音频同步。

我们需要定义一个`audio_clock`来记录音频播放的时钟

```c++
if (index == player->audio_stream_index) {
    player->audio_clock = packet->pts * av_q2d(stream->time_base);
    LOGD("SyncPlayer: Playing audio loop");
    audio_play(player, frame, env);
}
```

然后在视频播放的时候利用这个`audio_clock`进行一定的 delay

```c++
if (index == player->video_stream_index) {
    auto audio_clock = player->audio_clock;
    double timestamp;
    if (packet->pts == AV_NOPTS_VALUE) {
        timestamp = 0;
    } else {
        timestamp = frame->best_effort_timestamp * av_q2d(stream->time_base);
    }
    double frame_rate = av_q2d(stream->avg_frame_rate); // fps = 1 / stream->avg_frame_rate
    frame_rate += frame->repeat_pict * (frame_rate * 0.5); // repeat_dict代表当前frame必须delay的时间, extra_delay = repeat_pict / (2*fps)
    if (timestamp == 0.0) {
        usleep(frame_rate * 1000);
    } else {
        if (fabs(timestamp - audio_clock) > AV_SYNC_THRESHOLD_MIN
            && fabs(timestamp - audio_clock) < AV_NOSYNC_THRESHOLD) {
            if (timestamp > audio_clock) {
                usleep((unsigned long)((timestamp - audio_clock)*1000000));
            }
        }
    }
    video_play(player, frame, env);
}
```

至此，一个简单地音视频播放器就完成了.
