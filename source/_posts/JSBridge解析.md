---
title: JSBridge解析
date: 2019-12-21 20:10:53
tags: WebView, JSBridge
---

## 背景

利用 webview 的特性，我们设计一个 jsbridge 来实现 js 和 native 的**双向通信**

### native 和 JS 的调用实现手段

#### JavaScript 调用 Native

1. 注入 api
   这种方式的实现要么通过 webview 的借口，向 js 的 window 对象注入能力，js 调用这些能力时直接执行对应的 native 逻辑
   要么就是在我们实现浏览器的 jsbinding 时，就预埋一些基础能力(类似系统级别的函数接口)

2. 拦截 URL SCHEME

   这种实现方式主要依赖 native 的拦截能力，让 js 向一个 native 约定的 scheme 发送请求(例如使用 iframe.src)
   这种实现的缺陷一是会有 url 长度的隐患，另外创建请求会有耗时

#### Native 调用 JS

    相对简单，使用webview的evaluateJS类似的接口就可以调用

### JSBridge 实现方式

本质上讲，JSBridge 可以当作是一种 RPC，其核心实现就是句柄解析调用，JSBridge 的句柄往往就是一个 BridgeName，同时携带约定数据格式的序列化数据

JS 调用 native 的 RPC 实现是比较简单的，大致可以表达为:

```javascript
window.JSBridge = {
  // 调用 Native
  invoke: function(bridgeName, data) {
    // 判断环境，获取不同的 nativeBridge
    nativeBridge.postMessage({
      bridgeName: bridgeName,
      data: data || {}
    });
  },
  receiveMessage: function(msg) {
    var bridgeName = msg.bridgeName,
      data = msg.data || {};
    // 具体逻辑
  }
};
```

这里有个问题，目前的调用但是单向的，如何在一次调用实现双向通信呢？这里实际上可以类比 JSONP 机制进行处理

    JSONP请求发送时，url参数里有callback参数，其值是当前唯一，且被当前window记录，随后服务器返回的记录中，也会用这个参数值为句柄，调用相应的回调函数

因此我们可以参考这个设计，在我们的 jsbridge 设计一个 id 的生成器，和一个 id 保持一对一映射关系的 map，每次 js 调用 native 时，将这个 id 一并带给 native，native 不做理解，透传这个 id 回到 js 端，js 可以利用这个 id 在 map 中寻找到对应的 callback，调用对应的回调。用代码描述如下:

```javascript
(function () {
    var id = 0,
        callbacks = {};

    window.JSBridge = {
        // 调用 Native
        invoke: function(bridgeName, callback, data) {
            // 判断环境，获取不同的 nativeBridge
            var thisId = id ++; // 获取唯一 id
            callbacks[thisId] = callback; // 存储 Callback
            nativeBridge.postMessage({
                bridgeName: bridgeName,
                data: data || {},
                callbackId: thisId // 传到 Native 端
            });
        },
        receiveMessage: function(msg) {
            var bridgeName = msg.bridgeName,
                data = msg.data || {},
                callbackId = msg.callbackId; // Native 将 callbackId 原封不动传回
            // 具体逻辑
            // bridgeName 和 callbackId 不会同时存在
            if (callbackId) {
                if (callbacks[callbackId]) { // 找到相应句柄
                    callbacks[callbackId](msg.data); // 执行调用
                }
            } elseif (bridgeName) {

            }
        }
    };
})();
```

## P.S jsbridge 如何解决加载顺序问题？

根据我们之前的分析，jsbridge 的建立应该在业务的 js 执行前调用，那么这里就会有一个强逻辑的假设，我们有没有办法解除这种依赖关系呢？
jsbridge 的解决方案是业务的 js 将自己将要调用的 bridge 调用通过闭包的形式传到 window 的一个固定对象中，然后我们在加载 jsbridge 时，从这个约定的固定对象中将所有的调用闭包取出，然后依次调用即可

因为这层逻辑的存在，即使你的 jsbridge 因为网页的 CSP 策略导致加载 bridge 的请求被屏蔽，但由于屏蔽的请求仍旧会触发终端 native 的拦截器代码，你仍旧可以加载 jsbridge，同时利用闭包特性，将请求依次调用回来
