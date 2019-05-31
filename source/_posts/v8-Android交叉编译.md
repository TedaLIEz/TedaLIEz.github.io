---
title: v8 Android交叉编译
date: 2019-05-31 22:16:46
tags:
---

- [编译准备](#%E7%BC%96%E8%AF%91%E5%87%86%E5%A4%87)
- [编译](#%E7%BC%96%E8%AF%91)
- [Android端交叉编译](#android%E7%AB%AF%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91)
- [开始编译](#%E5%BC%80%E5%A7%8B%E7%BC%96%E8%AF%91)
- [Android 项目配置](#android-%E9%A1%B9%E7%9B%AE%E9%85%8D%E7%BD%AE)


## 编译准备

整个编译考虑到网络问题，强烈建议使用VPS的方式在服务器上进行编译。


## 编译

第一个难关就是代码的获取，由于v8整个工具链比较长，这里建议直接用Proxifier设置代理后下载. Proxifier的配置这里不赘述. 整个过程都需要代理

安装好Git和depot_tools后

```bash
mkdir ~/v8
cd ~/v8
fetch v8
cd v8

alias gm=/path/to/v8/tools/dev/gm.py

gm x64.release
```

参考 https://v8.dev/docs/build-gn 来做一些编译定制化



## Android端交叉编译

遵循上述步骤后, 来到`~/v8`你会看到一个.glient文件, 修改文件为

```yaml
 solutions = [
    {
        "url": "https://chromium.googlesource.com/v8/v8.git",
        "managed": False,
        "name": "v8",
        "deps_file": "DEPS",
        "custom_deps": {},
    },
];
target_os = ["android", "unix"];
```

之后`gclient sync`来获取Android交叉编译需要的工具

## 开始编译

这里我们使用gn工具进行编译，下面贴以下我的gn编译的配置

tools/dev/v8gen.py gen -m client.v8.ports -b “V8 Android Arm - builder” android.arm.release


```
android_unstripped_runtime_outputs = true
v8_use_external_startup_data = false
is_debug = false
symbol_level = 1
target_cpu = "arm"
target_os = "android"
use_goma = false
v8_enable_i18n_support = false
v8_static_library = true
is_component_build = false
v8_monolithic = true
v8_android_log_stdout = true
```

其中v8_monolithic表示将静态库编译为一个文件，而非几个单独文件

之后ninja -C android.arm.release

编译成功后，在android.arm.release/obj找到libv8_monolith.a

## Android 项目配置

我们需要把v8的头文件include进来，同时配置CMakeLists.txt

```
add_library(v8 STATIC IMPORTED)
set_target_properties( v8 PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/${ANDROID_ABI}/libv8_monolith.a)

add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        ${CMAKE_SOURCE_DIR}/src/main/cpp/native-lib.cpp)

target_include_directories( native-lib PRIVATE ${CMAKE_SOURCE_DIR}/libs/include)

```


branch_head/7.2分支上测试成功

具体的代码可以参考https://github.com/TedaLIEz/V8AndroidEmbedding

P.S

将7.4之后的v8编译，就会遇到一个错误

```
../../buildtools/third_party/libc++/trunk/include/string:2724: error: undefined reference to 'std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >::insert(unsigned int, char const*, unsigned int)'
/Users/JianGuo/AndroidProject/V8DemoApplication/app/src/main/cpp/native-lib.cpp:25: error: undefined reference to 'v8::platform::NewDefaultPlatform(int, v8::platform::IdleTaskSupport, v8::platform::InProcessStackDumping, std::__ndk1::unique_ptr<v8::TracingController, std::__ndk1::default_delete<v8::TracingController> >)'
clang++: error: linker command failed with exit code 1 (use -v to see invocation)
ninja: build stopped: subcommand failed.
```

看日志，我们似乎编译出了一个错误的命名空间。那么我们来检查一下我们的静态库的命名空间

` objdump -D app/libs/armeabi-v7a/libv8_monolith.a | grep NewDefault`

得到:
```
Disassembly of section .text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
_ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
       4:       81 b0 01 2b     blhs    #442884 <_ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE+0x6C210>
Disassembly of section .rel.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
.rel.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
Disassembly of section .ARM.exidx.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
.ARM.exidx.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
Disassembly of section .rel.ARM.exidx.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
.rel.ARM.exidx.text._ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE:
Disassembly of section .text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
_ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
Disassembly of section .rel.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
.rel.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
Disassembly of section .ARM.exidx.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
.ARM.exidx.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
Disassembly of section .rel.ARM.exidx.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:
.rel.ARM.exidx.text._ZN2v811ArrayBuffer9Allocator19NewDefaultAllocatorEv:

```

其中得到这么一串粉碎命名：

`_ZN2v88platform18NewDefaultPlatformEiNS0_15IdleTaskSupportENS0_21InProcessStackDumpingENSt3__110unique_ptrINS_17TracingControllerENS3_14default_deleteIS5_EEEE`

通过demangle后得到:

`v8::platform::NewDefaultPlatform(int, v8::platform::IdleTaskSupport, v8::platform::InProcessStackDumping, std::__1::unique_ptr<v8::TracingController, std::__1::default_delete<v8::TracingController> >)`
也就是说，我们的静态库编译出了std::__1::unique_ptr，但是ld时却寻找std::__ndk1::unique_ptr。

继续看日志，发现编译时尝试寻找`../../buildtools/third_party/libc++/trunk/include/string:2724`, 不妨去v8的源码中看一下。来到buildtools/third_party/libc++/trunk/__config:
```
#ifndef _LIBCPP_ABI_NAMESPACE
# define _LIBCPP_ABI_NAMESPACE _LIBCPP_CONCAT(__,_LIBCPP_ABI_VERSION)
#endif
#define _LIBCPP_BEGIN_NAMESPACE_STD namespace std { inline namespace _LIBCPP_ABI_NAMESPACE {
#define _LIBCPP_END_NAMESPACE_STD  } }
#define _VSTD std::_LIBCPP_ABI_NAMESPACE
_LIBCPP_BEGIN_NAMESPACE_STD _LIBCPP_END_NAMESPACE_STD
```
这里我们展开后其实就会得到:

```c++
namespace std {
    inline namespace __1 {

    }
}
```

对比以下ndk中的__config:

thrid_party/android_ndk/cxx-stl/llvm-libc++/include/__config

```c++
#define _LIBCPP_NAMESPACE _LIBCPP_CONCAT(__ndk,_LIBCPP_ABI_VERSION)
#define _LIBCPP_BEGIN_NAMESPACE_STD namespace std {inline namespace _LIBCPP_NAMESPACE {
#define _LIBCPP_END_NAMESPACE_STD  } }
#define _VSTD std::_LIBCPP_NAMESPACE

namespace std {
  inline namespace _LIBCPP_NAMESPACE {
  }
}
```
这里展开后为:

```c++
namespace std {
  inline namespace __ndk1 {
  }
}
```
这里显然是不一样的。也就是说ndk的libc++实际上更改了命名空间到__ndk这个前缀上，而v8进行编译时使用的libc++是
https://chromium.googlesource.com/chromium/llvm-project/libcxx
二者的命名空间显然是不同的。但为什么我编译使用的是buildtools/third_party/libc++而非thrid_party/android_ndk/cxx-stl/llvm-libc++？这个我还没有答案。
