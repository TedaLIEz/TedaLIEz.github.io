---
title: FFMpeg-Android开发-简单播放音频
date: 2020-08-29 09:04:55
tags:
---

本文部分对应的源代码可参考[https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.2](https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.2)

## 具体实现

音频播放的整个流程和视频非常相似，都经历了下面几个步骤

1. 解析 container
2. 根据 container，获得我们关心的媒体数据（例如播放视频，那我们只关心视频媒体）
3. 根据媒体信息获得对应的解码器
4. 将对应的数据送给解码器
5. 解码器解码，输出帧
6. 将帧渲染到目标区域(播放音频)

1-5 的步骤几乎可以参考播放视频的流程，这里简单看下不同

```c++

auto swr_context = swr_alloc();
auto out_buffer = (uint8_t *) av_malloc(44100 * 2);


// expected sample output
uint64_t out_channel_layout = AV_CH_LAYOUT_STEREO;
auto out_format = AV_SAMPLE_FMT_S16;

auto out_sample_rate = audio_codec_context->sample_rate;

// expected sample out para end

swr_alloc_set_opts(swr_context,
    out_channel_layout, out_format, out_sample_rate,
    audio_codec_context->channel_layout, audio_codec_context->sample_fmt, audio_codec_context->sample_rate,
0, nullptr);


swr_init(swr_context);

auto out_channels = av_get_channel_layout_nb_channels(AV_CH_LAYOUT_STEREO);
```

这一段的主要目的是配置音频的转码格式，跟视频不同，但基本逻辑一致

```c++
auto player_class = env->GetObjectClass(instance);
auto create_audio_track_method_id = env->GetMethodID(player_class, "createAudioTrack", "(II)V");
env->CallVoidMethod(instance, create_audio_track_method_id, 44100, out_channels);


auto play_audio_track_method_id = env->GetMethodID(player_class, "playAudioTrack", "([BI)V");



```

这里我们通过 JNI 来调用 Java 层的 AudioTrack 相关 API 来播放音频

至此，我们已经完成了音频的播放；接下来就剩下如何同时播放视频和音频，以及音视频同步问题了。下篇文章，我们就会来着手实现一个时间同步的简单播放器
