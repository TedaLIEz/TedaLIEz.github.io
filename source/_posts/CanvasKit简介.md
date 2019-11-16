---
title: CanvasKit简介
date: 2019-07-14 10:08:57
tags: WebAssembly
---

- [CanvasKit简介](#CanvasKit%E7%AE%80%E4%BB%8B)
  - [编译原理](#%E7%BC%96%E8%AF%91%E5%8E%9F%E7%90%86)
    - [编译](#%E7%BC%96%E8%AF%91)
    - [编译产物浅析](#%E7%BC%96%E8%AF%91%E4%BA%A7%E7%89%A9%E6%B5%85%E6%9E%90)
    - [编译脚本解析](#%E7%BC%96%E8%AF%91%E8%84%9A%E6%9C%AC%E8%A7%A3%E6%9E%90)
    - [小结](#%E5%B0%8F%E7%BB%93)
  - [可应用场景](#%E5%8F%AF%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF)
  - [小结](#%E5%B0%8F%E7%BB%93-1)

# CanvasKit简介

CanvasKit是以WASM为编译目标的Web平台图形绘制接口，其目标是将Skia的图形API导出到Web平台。
从代码提交记录来看，CanvasKit作为了一个Module放置在整个代码仓库中，最早的一次提交记录在2018年9月左右，是一个比较新的codebase

本文简单介绍一下Skia是如何编译为Web平台的，其性能以及未来的可应用场景


## 编译原理

整个canvaskit模块的代码量非常少:
```
.gitignore
CHANGELOG.md
Makefile
WasmAliases.h
canvaskit/  发布代码，canvaskit介绍文档
canvaskit_bindings.cpp
compile.sh  编译脚本
cpu.js
debug.js
externs.js
fonts/  字体资源文件
gpu.js
helper.js
htmlcanvas/
interface.js
karma.bench.conf.js
karma.conf.js
package.json
particles_bindings.cpp
perf/ 性能数据
postamble.js
preamble.js
ready.js
release.js
serve.py
skottie.js
skottie_bindings.cpp
tests/    测试代码
```

整个模块我们可以看到其实没有修改包括任何skia的代码文件，只是在编译时指明了skia的源码依赖，同时写了一些胶水代码，从这里可以看出skia迁移至WASM并没有付出很多额外的改造工作。


### 编译

设置好WASM工具链EmscriptenSDK的环境变量后运行compile.sh就会在`out`文件夹中得到`canvaskit.js`和`canvaskit.wasm`这两个编译产物，这里为了分析选择编译一个debug版本:

```
./compile.sh debug
```

debug版本会得到一个未混淆的canvaskit.js，方便我们分析其实现

### 编译产物浅析

为了快速了解整个模块的情况，直接观察canvaskit.js和canvaskit.wasm文件，先来看下canvaskit.js
js代码量比较大，这里摘取一段最能展示其运行原理的代码:
```javascript
function makeWebGLContext(canvas, attrs) {
    // These defaults come from the emscripten _emscripten_webgl_create_context
    var contextAttributes = {
    alpha: get(attrs, 'alpha', 1),
    depth: get(attrs, 'depth', 1),
    stencil: get(attrs, 'stencil', 0),
    antialias: get(attrs, 'antialias', 1),
    premultipliedAlpha: get(attrs, 'premultipliedAlpha', 1),
    preserveDrawingBuffer: get(attrs, 'preserveDrawingBuffer', 0),
    preferLowPowerToHighPerformance: get(attrs, 'preferLowPowerToHighPerformance', 0),
    failIfMajorPerformanceCaveat: get(attrs, 'failIfMajorPerformanceCaveat', 0),
    majorVersion: get(attrs, 'majorVersion', 1),
    minorVersion: get(attrs, 'minorVersion', 0),
    enableExtensionsByDefault: get(attrs, 'enableExtensionsByDefault', 1),
    explicitSwapControl: get(attrs, 'explicitSwapControl', 0),
    renderViaOffscreenBackBuffer: get(attrs, 'renderViaOffscreenBackBuffer', 0),
    };
    if (!canvas) {
    SkDebug('null canvas passed into makeWebGLContext');
    return 0;
    }
    // This check is from the emscripten version
    if (contextAttributes['explicitSwapControl']) {
    SkDebug('explicitSwapControl is not supported');
    return 0;
    }
    // GL is an enscripten provided helper
// See https://github.com/emscripten-core/emscripten/blob/incoming/src/library_webgl.js
return GL.createContext(canvas, contextAttributes);
}

CanvasKit.GetWebGLContext = function(canvas, attrs) {
return makeWebGLContext(canvas, attrs);
};

var GL= {
    // ...
    init:function () {
        GL.miniTempBuffer = new Float32Array(GL.MINI_TEMP_BUFFER_SIZE);
        for (var i = 0; i < GL.MINI_TEMP_BUFFER_SIZE; i++) {
          GL.miniTempBufferViews[i] = GL.miniTempBuffer.subarray(0, i+1);
        }
      },
      //...
      createContext:function (canvas, webGLContextAttributes) {
        var ctx = (canvas.getContext("webgl", webGLContextAttributes)
            || canvas.getContext("experimental-webgl", webGLContextAttributes));
        return ctx && GL.registerContext(ctx, webGLContextAttributes);
      },registerContext:function (ctx, webGLContextAttributes) {
        var handle = _malloc(8); // Make space on the heap to store GL context attributes that need to be accessible as shared between threads.
        assert(handle, 'malloc() failed in GL.registerContext!');
        var context = {
          handle: handle,
          attributes: webGLContextAttributes,
          version: webGLContextAttributes.majorVersion,
          GLctx: ctx
        };
        // Store the created context object so that we can access the context given a canvas without having to pass the parameters again.
        if (ctx.canvas) ctx.canvas.GLctxObject = context;
        GL.contexts[handle] = context;
        if (typeof webGLContextAttributes.enableExtensionsByDefault === 'undefined' || webGLContextAttributes.enableExtensionsByDefault) {
          GL.initExtensions(context);
        }
        return handle;
      },makeContextCurrent:function (contextHandle) {
        GL.currentContext = GL.contexts[contextHandle]; // Active Emscripten GL layer context object.
        Module.ctx = GLctx = GL.currentContext && GL.currentContext.GLctx; // Active WebGL context object.
        return !(contextHandle && !GLctx);
      },
      // ...
}
```

代码中出现了大量的WebGL指令和2d的绘制js代码，其实这一块就是EmscriptenSDK对OpenGL的胶水代码(https://emscripten.org/docs/porting/multimedia_and_graphics/OpenGL-support.html), 换言之，canvaskit的绘制代码没有脱离浏览器提供的webgl和context2d的相关接口，毕竟这也是目前在浏览器进行绘制操作的唯一途径

那编译的wasm文件做了啥呢？简单看一下对应wasm的一部分代码, 这也是一个比较庞大的文件，我们只关注一下wasm和js连接的桥梁代码:

```WebAssembly
 (import "env" "_eglGetCurrentDisplay" (func $_eglGetCurrentDisplay (result i32)))
 (import "env" "_eglGetProcAddress" (func $_eglGetProcAddress (param i32) (result i32)))
 (import "env" "_eglQueryString" (func $_eglQueryString (param i32 i32) (result i32)))
 (import "env" "_emscripten_glActiveTexture" (func $_emscripten_glActiveTexture (param i32)))
 (import "env" "_emscripten_glAttachShader" (func $_emscripten_glAttachShader (param i32 i32)))
 (import "env" "_emscripten_glBeginQueryEXT" (func $_emscripten_glBeginQueryEXT (param i32 i32)))
 (import "env" "_emscripten_glBindAttribLocation" (func $_emscripten_glBindAttribLocation (param i32 i32 i32)))
 (import "env" "_emscripten_glBindBuffer" (func $_emscripten_glBindBuffer (param i32 i32)))
 (import "env" "_emscripten_glBindFramebuffer" (func $_emscripten_glBindFramebuffer (param i32 i32)))
 (import "env" "_emscripten_glBindRenderbuffer" (func $_emscripten_glBindRenderbuffer (param i32 i32)))
 (import "env" "_emscripten_glBindTexture" (func $_emscripten_glBindTexture (param i32 i32)))
 (import "env" "_emscripten_glClear" (func $_emscripten_glClear (param i32)))
 (import "env" "_emscripten_glClearColor" (func $_emscripten_glClearColor (param f64 f64 f64 f64)))
 (import "env" "_emscripten_glClearDepthf" (func $_emscripten_glClearDepthf (param f64)))
 (import "env" "_emscripten_glCompileShader" (func $_emscripten_glCompileShader (param i32)))
 ...
```

这里省略了一部分，但是仍然可以看出，wasm对绘制的支持全部依赖其运行环境中js注入的函数实现

以这里的`_emscripten_glBindTexture`函数为例，对应到js为:

```javascript
var asmGlobalArg = {}

var asmLibraryArg = {
    "_emscripten_glBindTexture": _emscripten_glBindTexture // .... 上文wasm中的函数注册，这里略去，只保留_emscripten_glBindTexture
}
Module['asm'] = function(global, env, providedBuffer) {
    // memory was already allocated (so js could use the buffer)
    env['memory'] = wasmMemory
    ;
    // import table
    env['table'] = wasmTable = new WebAssembly.Table({
    'initial': 1155075,
    'maximum': 1155075,
    'element': 'anyfunc'
    });
    env['__memory_base'] = 1024; // tell the memory segments where to place themselves
    env['__table_base'] = 0; // table starts at 0 by default (even in dynamic linking, for the main module)

    var exports = createWasm(env);   // 加载WASM对象, env即WASM对象加载时所需要的上下文，包括内存大小，函数表，和堆栈起始地址
    assert(exports, 'binaryen setup failed (no wasm support?)');
    return exports;
};
// EMSCRIPTEN_START_ASM
var asm =Module["asm"]// EMSCRIPTEN_END_ASM
(asmGlobalArg, asmLibraryArg, buffer);
function _emscripten_glBindTexture(target, texture) {
    GL.validateGLObjectID(GL.textures, texture, 'glBindTexture', 'texture');
    GLctx.bindTexture(target, GL.textures[texture]);
}
```

GLctx通过代码我们也能找到对应:

```javascript
    createContext:function (canvas, webGLContextAttributes) {
        var ctx = (canvas.getContext("webgl", webGLContextAttributes)
            || canvas.getContext("experimental-webgl", webGLContextAttributes));
        return ctx && GL.registerContext(ctx, webGLContextAttributes);
    },registerContext:function (ctx, webGLContextAttributes) {
    var handle = _malloc(8); // Make space on the heap to store GL context attributes that need to be accessible as shared between threads.
    assert(handle, 'malloc() failed in GL.registerContext!');
    var context = {
        handle: handle,
        attributes: webGLContextAttributes,
        version: webGLContextAttributes.majorVersion,
        GLctx: ctx 
    };
    // Store the created context object so that we can access the context given a canvas without having to pass the parameters again.
    if (ctx.canvas) ctx.canvas.GLctxObject = context;
    GL.contexts[handle] = context;
    if (typeof webGLContextAttributes.enableExtensionsByDefault === 'undefined' || webGLContextAttributes.enableExtensionsByDefault) {
        GL.initExtensions(context);
    }
    return handle;
    },makeContextCurrent:function (contextHandle) {
  
        GL.currentContext = GL.contexts[contextHandle]; // Active Emscripten GL layer context object.
        Module.ctx = GLctx = GL.currentContext && GL.currentContext.GLctx; // Active WebGL context object.  // GLCtx变量
        return !(contextHandle && !GLctx);
    }
```

所以这里的bindTexture实际上就是WebGL的bindTexture指令(https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/bindTexture#Syntax)

分析到这里，我们可以得到一个基本结论: **canvaskit中绘制的实现全部在canvaskit.js中调用浏览器绘制API来实现，而计算相关的内容全部放在了wasm中实现**

### 编译脚本解析

通过对编译产物的分析，我们可以发现canvaskit绝大部分的绘制都是借助了Web API中的2d或webgl绘制API来完成的。这里需要分析的是canvaskit如何搭建了skia原生绘制代码和浏览器绘制API的桥梁。

看到compile.sh发现最后一句话涉及到很多canvaskit目录下的文件，因此直接结合编译日志的相关内容分析。

其他的日志都是常规的skia编译命令，只不过执行程序换成了em++而已，em++就是EmscriptenSDK中的编译器命令，可以类比为g++，这些命令会把skia编译为几个静态库

我们略过之前的skia编译命令来到最后一段，这是真正生成WASM产物的地方，其中有大量的逻辑是涉及到canvaskit中的胶水代码的。略去链接, 编译器优化设置, Skia静态库路径的指定, Skia宏定义和头文件路径指定，我们将会得到:

```shell script
/Users/JianGuo/VSCodeProject/emsdk/emscripten/1.38.28/em++  \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/debug.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/cpu.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/gpu.js \
--bind \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/preamble.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/helper.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/interface.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/skottie.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/preamble.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/util.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/color.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/font.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/canvas2dcontext.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/htmlcanvas.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/imagedata.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/lineargradient.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/path2d.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/pattern.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/radialgradient.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/htmlcanvas/postamble.js \
--pre-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/postamble.js \
--post-js /Users/JianGuo/Desktop/skia/skia/modules/canvaskit/ready.js \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/fonts/NotoMono-Regular.ttf.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/canvaskit_bindings.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/particles_bindings.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/skottie_bindings.cpp \
modules/skottie/utils/SkottieUtils.cpp \
-s ALLOW_MEMORY_GROWTH=1 \ # 允许申请比TOTAL_MEMORY更大的内存
-s EXPORT_NAME=CanvasKitInit \ # js中Module的名字
-s FORCE_FILESYSTEM=0 \ # 开启文件系统支持，用于js中对native的文件系统进行模拟
-s MODULARIZE=1 \ #启用Module的方式生成js，开启后编译的js产物将拥有一个Module作用域，而非全局作用域
-s NO_EXIT_RUNTIME=1 \ # 禁止使用exit函数
-s STRICT=1 \ # 确保编译器不使用弃用语法
-s TOTAL_MEMORY=128MB \ # WASM分配的总内存，如果比此内存更大的场景就需要扩展堆大小
-s USE_FREETYPE=1 \ # 使用emscripten-ports导出的freetype库
-s USE_LIBPNG=1 \ # 使用emscripten-ports导出的libpng库
-s WARN_UNALIGNED=1 \ # 编译时警告未对齐(align)
-s USE_WEBGL2=0 \ # 不使用WebGL2
-s WASM=1 \ # 编译为WASM
-o out/canvaskit_wasm_debug/canvaskit.js # 指定编译路径
```

其中，`pre-js <file>`表示将指定文件的内容插入到生成的js文件前, `post-js`表示将指定文件的内容插入到生成的js文件后，我们以`skia/modules/canvaskit/htmlcanvas/htmlcanvas.js`为例，看看这些插入的文件都干了啥:

```javascript
CanvasKit.MakeCanvas = function(width, height) {
  var surf = CanvasKit.MakeSurface(width, height);
  if (surf) {
    return new HTMLCanvas(surf);
  }
  return null;
}

function HTMLCanvas(skSurface) {
  this._surface = skSurface;
  this._context = new CanvasRenderingContext2D(skSurface.getCanvas());
  this._toCleanup = [];
  this._fontmgr = CanvasKit.SkFontMgr.RefDefault();

  // Data is either an ArrayBuffer, a TypedArray, or a Node Buffer
  this.decodeImage = function(data) {
    // ...
  }

  this.loadFont = function(buffer, descriptors) {
    //...
  }

  this.makePath2D = function(path) {
    //...
  }

  // A normal <canvas> requires that clients call getContext
  this.getContext = function(type) {
    //...
  }

  this.toDataURL = function(codec, quality) {
    //...
  }

  this.dispose = function() {
    //...
  }
}
```

其实就是对齐了一下浏览器实现，同时对齐了一下Skia内部的接口而已。
最后我们还剩下一段没有分析:

```shell script
/Users/JianGuo/VSCodeProject/emsdk/emscripten/1.38.28/em++  \
...
--bind \
...
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/fonts/NotoMono-Regular.ttf.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/canvaskit_bindings.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/particles_bindings.cpp \
/Users/JianGuo/Desktop/skia/skia/modules/canvaskit/skottie_bindings.cpp \
modules/skottie/utils/SkottieUtils.cpp \
...
```

根据文档，这段命令要求em++以Embind(https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#embind)连接C++代码和JS代码, embind简单来说就是emscriptenSDK提供的将C/C++代码暴露给JavaScript的便捷能力。这里不做重点介绍，我们直接看canvaskit用到的一个代码:

`particles_bindings.cpp`:

```c++
// ...
#include <emscripten.h>
#include <emscripten/bind.h>

using namespace emscripten;

EMSCRIPTEN_BINDINGS(Particles) {
    class_<SkParticleEffect>("SkParticleEffect")
        .smart_ptr<sk_sp<SkParticleEffect>>("sk_sp<SkParticleEffect>")
        .function("draw", &SkParticleEffect::draw, allow_raw_pointers())
        .function("start", select_overload<void (double, bool)>(&SkParticleEffect::start))
        .function("update", select_overload<void (double)>(&SkParticleEffect::update));

    function("MakeParticles", optional_override([](std::string json)->sk_sp<SkParticleEffect> {
        static bool didInit = false;
        if (!didInit) {
            REGISTER_REFLECTED(SkReflected);
            SkParticleAffector::RegisterAffectorTypes();
            SkParticleDrawable::RegisterDrawableTypes();
            didInit = true;
        }
        SkRandom r;
        sk_sp<SkParticleEffectParams> params(new SkParticleEffectParams());
        skjson::DOM dom(json.c_str(), json.length());
        SkFromJsonVisitor fromJson(dom.root());
        params->visitFields(&fromJson);
        return sk_sp<SkParticleEffect>(new SkParticleEffect(std::move(params), r));
    }));
    constant("particles", true);

}

```

上面代码经过em++编译后会直接将其功能内嵌进wasm文件中。至此，整个编译流程就分析完了

### 小结

这里用一张图来总结一下整个canvaskit的编译流程, 图中省去了编译器优化和js优化的流程:

```plantuml
partition em++ {
    (*) --> "libskottie.a, libsksg.a, libparticles.a, libskshaper.a, libharfbuzz.a, libicu.a, libskia.a"
    --> ===B1===
    --> "--bind modules/canvaskit/fonts/NotoMono-Regular.ttf.cpp
    --bind modules/canvaskit/canvaskit_bindings.cpp
    --bind modules/canvaskit/particles_bindings.cpp
    --bind modules/canvaskit/skottie_bindings.cpp
    --bind modules/skottie/utils/SkottieUtils.cpp"
    --> ===B2===

    ===B2=== -> "canvaskit.wasm"
    -->===B3===
    ===B2=== --> "--pre-js modules/canvaskit/htmlcanvas/*.js
    --pre-js modules/canvaskit/postamble.js
    --pre-js modules/canvaskit/preamble.js
    --pre-js modules/canvaskit/debug.js
    --pre-js modules/canvaskit/cpu.js
    --pre-js modules/canvaskit/gpu.js
    --pre-js modules/canvaskit/helper.js
    --pre-js modules/canvaskit/interface.js
    --pre-js modules/canvaskit/skottie.js"

    ===B1=== --> "--pre-js modules/canvaskit/htmlcanvas/*.js
    --pre-js modules/canvaskit/postamble.js
    --pre-js modules/canvaskit/preamble.js
    --pre-js modules/canvaskit/debug.js
    --pre-js modules/canvaskit/cpu.js
    --pre-js modules/canvaskit/gpu.js
    --pre-js modules/canvaskit/helper.js
    --pre-js modules/canvaskit/interface.js
    --pre-js modules/canvaskit/skottie.js"
    --> "--post-js modules/canvaskit/ready.js"
    --> "canvaskit.js"
    -->===B3===
    -->(*)
}

```

## 可应用场景

根据官方文档(https://skia.org/user/modules/canvaskit), canvaskit基于skia的API设计向web平台提供了更加方便的图形接口，可以说起到了类似GLWrapper的作用。

得益于Skia本身的其他扩展功能，canvaskit相比于浏览器原生绘制能力，支持了许多更加上层的业务级别功能，例如skia的动画模块skottie(https://skia.org/user/modules/skottie)

Skia中的skottie本身就支持Lottie动画解析和播放，由于Skia良好的跨平台能力，Android和iOS平台现在均可以使用Skia框架来播放Lottie动画，canvaskit则运用WebAssembly的技术来将跨平台的范围扩展到web上，使得web平台可以通过canvaskit的skottie相关接口直接播放lottie动画

对于Web应用而言，canvaskit提供了开发者更加友好的图形接口，并提供了常见的图形概念（例如Bitmap，Path等），减少了上层应用开发者对于绘制接口的理解负担，开发者只需要理解Skia的图形概念即可开发图形界面，有了skia他们也不需要理解复杂的webgl指令。


## 小结

得益于WASM的理念和EmscriptenSDK的能力，越来越多的native库可以直接导出web上供开发者使用。CanvasKit可以说是C++ Library向Web平台迁移的又一最佳实践。EmscriptenSDK已经做到将Skia这种规模的C++项目以WASM的方式迁移至Web平台，并保证其代码功能的一致性。整个迁移的过程的代价也就是编译工具链的替换和一部分胶水代码。