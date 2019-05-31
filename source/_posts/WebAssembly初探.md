---
title: WebAssembly初探
date: 2019-05-25 15:56:15
tags: WebAssembly
---
- [开发环境演进](#%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83%E6%BC%94%E8%BF%9B)
- [开发环境](#%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83)
  - [Emscripten工具链简述](#emscripten%E5%B7%A5%E5%85%B7%E9%93%BE%E7%AE%80%E8%BF%B0)
- [实战](#%E5%AE%9E%E6%88%98)
  - [JS中加载wasm的方式](#js%E4%B8%AD%E5%8A%A0%E8%BD%BDwasm%E7%9A%84%E6%96%B9%E5%BC%8F)
  - [JS中调用wasm中的函数或类](#js%E4%B8%AD%E8%B0%83%E7%94%A8wasm%E4%B8%AD%E7%9A%84%E5%87%BD%E6%95%B0%E6%88%96%E7%B1%BB)
    - [调用函数](#%E8%B0%83%E7%94%A8%E5%87%BD%E6%95%B0)
      - [原生实现](#%E5%8E%9F%E7%94%9F%E5%AE%9E%E7%8E%B0)
      - [Embind能力实现](#embind%E8%83%BD%E5%8A%9B%E5%AE%9E%E7%8E%B0)
      - [cwrap或ccall实现调用](#cwrap%E6%88%96ccall%E5%AE%9E%E7%8E%B0%E8%B0%83%E7%94%A8)
    - [引用一个类](#%E5%BC%95%E7%94%A8%E4%B8%80%E4%B8%AA%E7%B1%BB)

## 开发环境演进

目前稳定的webassembly的编译方式都需要通过asm.js来生成wasm(hacking)。LLVM WebAssembly 后端目前还处在研发过程中，没有稳定的版本。

![](WebAssemblyToolchain.png)

整个开发环境的演变可以参考下面这两个ppt
https://kripken.github.io/talks/emwasm.html

http://kripken.github.io/talks/wasm.html

## 开发环境
目前最常用的开发工具链是emscripten，emscripten可以将LLVM bitcode编译为javascript(LLVM to web compiler)

使用Emscripten可以:
1. 编译C和C++代码为javascript
2. 将任何可以编译为LLVM bitcode的代码编译为javascript
3. 编译C/C++ runtimes 代码为javascript

整个工具已经有了许多大型应用，可以认为是比较稳定的工具链 https://github.com/emscripten-core/emscripten/wiki/Porting-Examples-and-Demos


Emscripten同时也支持WebAssembly，默认通过asm2wasm将asm.js转为wasm https://github.com/WebAssembly/binaryen#cc-source--asm2wasm--webassembly

Emscripten同时也提供了一些标准库的实现和WebAssembly能力的封装，减少我们配置WASM的负担。当然你也可以选择不使用Emscripten提供的能力，采用最原始的方式操作WASM

### Emscripten工具链简述

![](EmscriptenToolchain.png)

emcc使用clang将C/C++转为LLVMbitcode，然后使用Emscripten自己的编译器后端Fastcomp将bitcode转为javascript https://emscripten.org/docs/compiling/WebAssembly.html#llvm-webassembly-backend


## 实战

```
Aside: When you compile C/C++ normally, you link with the system to provide implementations of the standard library methods your code uses.

JavaScript doesn't have these methods—either not with the same signatures or names (e.g, Math.atan in JavaScript vs atan in C), or because it's conceptually different (think malloc vs JavaScript's objects and garbage collection)—so Emscripten has to provide them for you.
```

ONLY_MY_CODE 标志位用来表示你在编译不需要emcc提供的一些stardard库函数

但是一般我们都是需要emcc提供的标准库函数的，不然我们连下面这个hello world代码都编译不出来

```c++
#include <iostream>

int main(int, char**) {
    std::cout << "Hello, world!\n";
    return 0;
}
```

用`emcc -o output.js *.cpp -s WASM=1 -s ONLY_MY_CODE=1`会提示:

```shell script
error: undefined symbol: _ZNKSt3__26locale9use_facetERNS0_2idE
warning: To disable errors for undefined symbols use `-s ERROR_ON_UNDEFINED_SYMBOLS=0`
error: undefined symbol: _ZNKSt3__28ios_base6getlocEv
error: undefined symbol: _ZNSt3__212basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEmc
error: undefined symbol: _ZNSt3__212basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED1Ev
error: undefined symbol: _ZNSt3__213basic_ostreamIcNS_11char_traitsIcEEE6sentryC1ERS3_
error: undefined symbol: _ZNSt3__213basic_ostreamIcNS_11char_traitsIcEEE6sentryD1Ev
error: undefined symbol: _ZNSt3__26localeD1Ev
error: undefined symbol: _ZNSt3__28ios_base5clearEj
error: undefined symbol: _ZSt9terminatev
error: undefined symbol: __cxa_begin_catch
error: undefined symbol: __gxx_personality_v0
error: undefined symbol: strlen
error: undefined symbol: _ZNSt3__24coutE
error: undefined symbol: _ZNSt3__25ctypeIcE2idE
Error: Aborting compilation due to previous errors
```

理由也很简单，因为iostream中用到的标准api没有找到实现


EXPORTED_FUNCTIONS表示我们希望从js能访问的函数，这个宏一般是我们编译希望不依赖Emscripten library的代码时用到。如果用了Emscripten提供的能力，不需要在编译参数里声明这个参数

### JS中加载wasm的方式

```javascript
async function createWebAssembly(path, importObject) {
    const result = await window.fetch(path);
    const bytes = await result.arrayBuffer();
    return WebAssembly.instantiate(bytes, importObject);
}
```

这里有个很关键的概念，importObject

API文档对这个对象的定义是一个包含需要被import的对象，可以理解为被加载wasm需要的上下文，而这个上下文是非常基础的，可以包含内存大小的定义。

一个典型的importObject如下

```javascript
const memory = new WebAssembly.Memory({initial: 256, maximum: 256});
const env = {
      'abortStackOverflow': _ => { throw new Error('overflow'); },
      'table': new WebAssembly.Table({initial: 0, maximum: 0, element: 'anyfunc'}),  // wasm调用JS里的方法的函数表
      '__table_base': 0,
      'memory': memory,
      '__memory_base': 1024,
      'STACKTOP': 0,
      'STACK_MAX': memory.buffer.byteLength,
};
const importObject = {env};
```

可以看到这个importObject配置了内存相关的内容，wasm的内存模型也是一个线性模型, 配置项配置了整个内存的大小，并区分了栈和堆的内存地址


上面的方式下，你不能用C/C++标准库的API，但好在Emscripten能够提供这些标准库的实现。Emscripten能够提供下面的能力：
1. 标准库方法，例如malloc，free等
2. 利用Emscripten Embind的能力将C++的函数和类暴露到js中

### JS中调用wasm中的函数或类

#### 调用函数

##### 原生实现

这种方式下我们不需要emscripten提供的能力来调用到wasm中的函数

```html
const wa = await createWebAssembly('output.wasm', importObject);
wa.instance.exports.fun();
```

##### Embind能力实现
emcc编译时默认会生成一个wasm文件和一个js文件，这个js文件就提供了相关的绑定能力，而我们在C++代码中则需要使用EMSCRIPTEN_BINDINGS这个宏

例如：
```c++
#include <emscripten/bind.h>

emscripten::val mandelbrot(int w, int h, double zoom, double moveX, double moveY);

EMSCRIPTEN_BINDINGS(hello)
{
        emscripten::function("mandelbrot", &mandelbrot);
}
```

我们可以在js中调用函数mandelbrot
```
<script src="mandelbrot.js"></script>
<script>
const mandelbrot = Module.mandelbrot(width, 1, -0.5, 0);
</script>
```

##### cwrap或ccall实现调用

这种方式依赖cwrap这个emscripten的api来实现


```c
#include "emscripten.h"
#include <stdlib.h>

EMSCRIPTEN_KEEPALIVE
uint8_t* create_buffer(int width, int height) {
  return malloc(width * height * 4 * sizeof(uint8_t));
}
```

编译时使用命令`emcc -s WASM=1 -s EXTRA_EXPORTED_RUNTIME_METHODS='["create_buffer"]' lib.c`

```html
<script src="lib.js"></script>
<script>
Module.onRuntimeInitialized = _ => {
    const fib = Module.cwrap('create_buffer', 'number', ['number']);
    console.log(fib(12));
  };
</script>
```

`EMSCRIPTEN_KEEPALIVE`表示提示编译器在优化过程中不要删去函数的代码，因为这个函数后续会被js调用


#### 引用一个类

利用Embind，在JS代码中引用c++里的一个类

```c++
#include <emscripten/bind.h>
class Mandelbrot {
public:
emscripten::val nextTile();
Mandelbrot(int width, int height, double zoom, double moveX, double moveY)
      : width(width), height(height), zoom(zoom), moveX(moveX), moveY(moveY) {}
};

EMSCRIPTEN_BINDINGS(hello) {
  emscripten::class_<Mandelbrot>("Mandelbrot")
      .constructor<int, int, double, double, double>()
      .function("nextTile", &Mandelbrot::nextTile);
}

```


```html
<script src="mandelbrot.js"></script>
<script>
const mandelbrot = new Module.Mandelbrot(width, height, 
    1, -0.5, 0);
const tile = mandelbrot.nextTile();
</script>
```


