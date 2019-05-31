---
title: NDK学习笔记
date: 2019-05-25 15:48:48
tags: NDK
---

记录一下自己NDK开发过程中零散的学习记录

## 如何让CMake下的NDK build打开verbose开关

```groovy
externalNativeBuild {
  cmake {
    cppFlags "-std=c++11"
    arguments "-DCMAKE_VERBOSE_MAKEFILE:BOOL=ON"   // 关键
  }
}

```

## 如何在externalNativeBuild中传递linker flags

The `-Wl,xxx` option for gcc passes a comma-separated list of tokens as a space-separated list of arguments to the linker. So

```
gcc -Wl,aaa,bbb,ccc
```

eventually becomes a linker call

```
ld aaa bbb ccc
```

## target_link_libraries

The idea is that you build modules in CMake, and link them together. Let's ignore header files for now, as they can be all included in your source files.

Say you have file1.cpp, file2.cpp, main.cpp. You add them to your project with:

```
ADD_LIBRARY(LibsModule 
    file1.cpp
    file2.cpp
)
```
Now you added them to a module called LibsModule. Keep that in mind. Say you want to link to pthread for example that's already in the system. You can combine it with LibsModule using the command:

```
target_link_libraries(LibsModule -lpthread)
```

And if you want to link a static library to that too, you do this:

```
target_link_libraries(LibsModule liblapack.a)
```

And if you want to add a directory where any of these libraries are located, you do this:

```
target_link_libraries(LibsModule -L/home/user/libs/somelibpath/)
```

Now you add an executable, and you link it with your main file:

```
ADD_EXECUTABLE(MyProgramExecBlaBla main.cpp)
```

(I added BlaBla just to make it clear that the name is custom). And then you link LibsModule with your executable module MyProgramExecBlaBla

```
target_link_libraries(MyProgramExecBlaBla LibsModule)
```

And this will do it.

What I see in your CMake file is a lot of redundancy. For example, why do you have texture_mapping, which is an executable module in your include directories? So you need to clean this up and follow the simple logic I explained. Hopefully it works.

In summary, it looks like this:

```
project (MyProgramExecBlaBla)  #not sure whether this should be the same name of the executable, but I always see that "convention"
cmake_minimum_required(VERSION 2.8)

ADD_LIBRARY(LibsModule 
    file1.cpp
    file2.cpp
)

target_link_libraries(LibsModule -lpthread)
target_link_libraries(LibsModule liblapack.a)
target_link_libraries(LibsModule -L/home/user/libs/somelibpath/)
ADD_EXECUTABLE(MyProgramExecBlaBla main.cpp)
target_link_libraries(MyProgramExecBlaBla LibsModule)
```

The most important thing to understand is the module structure, where you create modules and link them all together with your executable. Once this works, you can complicate your project further with more details. Good luck!