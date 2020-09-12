---
title: FFMpeg Android开发101-编译和引入
date: 2020-08-15 10:15:51
tags:
---

本系列会尝试将FFMpeg引入到android项目中，并借助FFMpeg完成一些音视频的简单饮用；FFMpeg作为一个成熟的音视频编解码工具被大量项目使用，但将FFMpeg引入到Android开发的文档并不多，国内有一部分但大量已经过时，这个系列会重新尝试带领大家完成整个过程。

本文对应的源代码请参考[https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.0](https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.0)


```
git clone https://github.com/TedaLIEz/MyFFMpegAndroid/tree/v1.0

cd MyFFMpegAndroid
git submodule update

cd ffmpeg-android-maker
export ANDROID_SDK_HOME=${YOUR_SDK_HOME}
export ANDROID_NDK_HOME=${YOUR_NDK_HOME}
./ffmpeg-android-maker.sh
```

之后用Android Studio打开项目，即可运行

## 前言

总的来说，将FFMpeg引入到Android项目开发分为下面几个步骤：

1. 使用Android NDK编译FFMpeg项目
2. 在自己的项目部署FFMpeg
3. 开始FFMpeg开发，验证效果

### 使用Android NDK编译FFMpeg

    TL;DR 参考https://github.com/Javernaut/ffmpeg-android-maker


首先需要简单了解些FFMpeg编译产物的一些职责:

`libavformat`: 处理文件container, stream
`libavcodec`: 编解码
`libswscale`: 图像处理
`libavutil`: util库

还有一些其他的库，可以自行参考FFmpeg文档

考虑到我们是编译一个Android的FFMpeg库，本质上这就是一个交叉编译，因此我简单介绍一下FFmpeg编译到Android上的一些配置


#### 编译配置

由于Android有着ARM, x86的32，64位处理器架构，因此我们需要编译4种二进制文件；这时需要一个交叉编译器(cross-compiler)来帮我们处理问题。

第二，编译过程不只是compile，还包括链接(linker)等其他工具, 我们统称为binutils

第三，我们需要Android系统自己的一些库和头文件，这些文件的存储位置我们成为sysroot


因此，整个编译需要的工具链包括`cross-compiler`, `binutils`, `sysroot`


那么我们怎么获得这些工具呢？答案显然就是在Android NDK中，如果之前你有查阅过Android-NDK的目录，那么你就会发现NDK的结构其实已经说明了这三个工具的划分



这里我们看一下configure的相关参数


```bash
./configure \
--prefix=${BUILD_DIR}/${ABI} \
--enable-cross-compile \
--target-os=android \
--arch=${TARGET_TRIPLE_MACHINE_BINUTILS} \
--sysroot=${SYSROOT} \
--cross-prefix=${CROSS_PREFIX} \
--cc=${CC} \
--extra-cflags="-O3 -fPIC" \
--enable-shared \
--disable-static \
${EXTRA_BUILD_CONFIGURATION_FLAGS} \
```


`prefix`指明了产物的路径

`target-os=android`: 指明我们的编译操作系统是Android

`--arch=${TARGET_TRIPLE_MACHINE_BINUTILS}`: 指明arm, aarch64, i686 and x86_64

`--sysroot=${SYSROOT}`: 指明sysroot的路径，这个路径一定是在Android NDK的路径下的一个子目录

`--cross-prefix=${CROSS_PREFIX}`: 指明binutils的工具名前缀，这个名字会追加到ld前当作linker使用，例如

```
armeabi-v7a: $TOOLCHAIN_PATH/bin/arm-linux-androideabi-
arm64-v8a:   $TOOLCHAIN_PATH/bin/aarch64-linux-android-
x86:         $TOOLCHAIN_PATH/bin/i686-linux-android-
x86_64:      $TOOLCHAIN_PATH/bin/x86_64-linux-android-
```

`--cc=${CC}`: 指明编译器, 例如

```
armeabi-v7a: $TOOLCHAIN_PATH/bin/armv7a-linux-androideabi16-clang
arm64-v8a:   $TOOLCHAIN_PATH/bin/aarch64-linux-android21-clang
x86:         $TOOLCHAIN_PATH/bin/i686-linux-android16-clang
x86_64:      $TOOLCHAIN_PATH/bin/x86_64-linux-android21-clang

```

`--enable-shard`和`--disable-static`表明我们是编译一个动态库

`--extra-cflags=”-O3 -fPIC”`表明了其他的C flag, `-O3`表明编译器优化级别, `-fPIC`是编译Android上的动态库必须的参数



#### FFMpeg自身的相关配置

除去上面编译工具链需要的参数外，我们还可以定制化我们的FFMpeg编译，例如:


```
--disable-runtime-cpudetect \
--disable-programs \
--disable-muxers \
--disable-encoders \
--disable-avdevice \
--disable-postproc \
--disable-swresample \
--disable-avfilter \
--disable-doc \
--disable-debug \
--disable-pthreads \
--disable-network \
--disable-bsfs \
--disable-decoders \
${DECODERS_TO_ENABLE}

```

这里`--disable`-xxx表示不需要FFMpeg的具体模块，这个可以根据自身app的开发来定制

上述工作全部完成后，就可以开始make && make install了

### 在Android项目中引入FFmpeg

编译完成后，你应该能获得如下的编译结果


```

.
├── include
│   ├── arm64-v8a
│   ├── armeabi-v7a
│   ├── x86
│   └── x86_64
└── lib
    ├── arm64-v8a
    ├── armeabi-v7a
    ├── x86
    └── x86_64
```


下一步就是引入到Android项目中了，首先在你的app/build.gradle中作如下修改


```groovy
android {
    ...
    defaultConfig {
        ...
        ndk {
            abiFilters 'x86', 'x86_64', 'armeabi-v7a', 'arm64-v8a'
        }
    }
    sourceSets {
        main {
            // let gradle pack the shared library into the apk
            jniLibs.srcDirs = ['../ffmpeg-android-maker/output/lib']
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

分别解释一下:

`abiFilters`: 指定了app支持的架构

`jniLibs.srcDirs`: 指定了app需要的动态库路径，android编译时会自动将这里指定的动态库打包进apk

`externalNativeBuild`这里指定了我们的`CMakeLists.txt`路径

下一步就是配置CMakeLists

```cmake
cmake_minimum_required(VERSION 3.4.1)  # 指定Cmake最低版本

set(ffmpeg_dir ${CMAKE_SOURCE_DIR}/../ffmpeg-android-maker/output)  # 设置ffmpeg_dir变量
include_directories(${ffmpeg_dir}/include/${ANDROID_ABI})  # 设置需要include的头文件路径，注意这里的ANDROID_ABI代表了在gradle中指定的abiFilters的每一个变量
set(ffmpeg_libs ${ffmpeg_dir}/lib/${ANDROID_ABI}) # 设置ffmpeg_libs变量，指明shared library路径



add_library(avutil SHARED IMPORTED) # 声明avutil库
set_target_properties(avutil PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libavutil.so) # 指定avutil库shared library路径

add_library(avformat SHARED IMPORTED) # 类似上面的声明
set_target_properties(avformat PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libavformat.so)

add_library(avfilter SHARED IMPORTED)
set_target_properties(avfilter PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libavfilter.so)

add_library(avcodec SHARED IMPORTED)
set_target_properties(avcodec PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libavcodec.so)

add_library(swscale SHARED IMPORTED)
set_target_properties(swscale PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libswscale.so)

add_library(swresample SHARED IMPORTED)
set_target_properties(swresample PROPERTIES IMPORTED_LOCATION ${ffmpeg_libs}/libswresample.so)

find_library(log-lib log) # 使用Android 的native log库，命名为log-lib
find_library(jnigraphics-lib jnigraphics) # 同上，使用jnigraphics

add_library(test_ffmpeg
        SHARED
        src/main/cpp/video_config.cpp
        src/main/cpp/video_config_jni.cpp
        src/main/cpp/utils.cpp
        src/main/cpp/main.cpp
        src/main/cpp/player.cpp)  # 添加我们自己项目中的代码，并命名为test_ffmpeg库

target_link_libraries(
        test_ffmpeg
        ${log-lib}
        ${jnigraphics-lib}
        android
        avformat
        avcodec
        swscale
        avutil
        swresample
        avfilter) # 通知linker将上述所有library链接

```


具体含义已经在注释中

全部配置完成后，可以尝试进行编译；剩下的问题就是JNI相关的知识和Android自身的相关开发知识了，本文就不再赘述。需要注意的一个小点就是在Java层System.loadLibrary时需要注意加载顺序


```kotlin
    init {
            listOf("avutil", "avcodec", "avformat", "swscale", "test_ffmpeg").forEach {
                System.loadLibrary(it)
            }
        }
```


之后运行app，如果没有出现crash，就基本证明我们的引入是ok的了。Have Fun！