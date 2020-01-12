---
title: Kotlin协程浅析
date: 2020-01-05 19:00:55
tags: kotlin, coroutine
---

本文主要参考 KotlinConf 2017 内容来对 Kotlin 中协程的实现进行解析

TL;DR;

    Kotlin的协程本质上仍旧是Callback式的调用, 但采用了CPS的编程范式，使得我们的代码显得线性

这里略过对协程的定义，首先我们需要解释 CPS 是什么东西

## CPS()

CPS 实际上是一种编程风格，根据 wiki 定义

    In functional programming, continuation-passing style (CPS) is a style of programming in which control is passed explicitly in the form of a continuation. (from Wikipedia.)

举个简单的例子，如果我们定义一类型的函数，这些函数都以 async 开头，并且在函数参数中都带有一个 callback 函数参数，每个函数在执行自己的逻辑完成后，都通过 callback 来处理结果

```kotlin

fun asyncDoSomething(param1: Int, param2: Any, callback: (Any?) -> Unit) {
    // ...
    // when execution is done
    callback.invoke(result)
}

```

这种类型的函数中，callback 往往被称作 continuation，因为实际上它表达的是下一个函数执行的逻辑，即这个回调函数决定了程序接下来的行为

上述这种方式就是一种标准的 CPS 实现方式

举一个现实例子来说，例如:

```kotlin
fun postItem(item: Item) {
    val token = requestToken()
    val post = createPost(token, item)
    processPost(post)
}
```

这里你有一个过程需要三个子过程来完成

对于 requestToken()函数来说，createPost 和 processPost 是其过程的 continuation

那么我们可以尝试把 postItem 这个过程转换为一个 CPS 式的编码范式

```kotlin
fun postItem(item: Item) {
    requestToken { token ->
        createPost(token, item) { post ->
            processPost(post)
        }
    }
}
```

这种写法看上去还算 OK，但是也有很多问题，例如:

1. Callback Hell
2. 过分多的回调，会带来非常大的调用栈开销

那么 Kotlin 是如何实现 CPS 的呢？

## Kotlin 对 CPS 的实现

Kotlin 对 CPS 风格的实现，实际上就是解决了 Kotlin 对协程的支持

```kotlin
suspend fun createPost(token: Token, item: Item) : Post { ... }
```

实际上会转换为:

```kotlin
Object createPost(Token token, Item item, Continuation<Post> cont) { … }

interface Continuation<in T> {
    val context: CoroutineContext
    fun resume(value: T)
    fun resumeWithException(exception: Throwable)
}
```

可以很明显的发现 kotlin 的 Continuation 对象就是一个典型的 Callback

接下来我们来看看 Kotlin 对协程的实现方式

## Kotlin 对协程的实现

```kotlin
suspend fun postItem(item: Item) {
    val token = requestToken()
    val post = createPost(token, item)
    processPost(post)
}

```

上面是一段典型的 kotlin 协程代码，根据之前的讨论，对于协程，我们有一种典型的实现:

```kotlin
fun postItem(item: Item) {
    requestToken { token ->
        createPost(token, item) { post ->
            processPost(post)
        }
    }
}
```

这种实现被认为存在回调地狱和栈开销的问题，接下来我们看看 kotlin 是如何解决的

```kotlin
fun postItem(item: Item, cont: Continuation) {
    val sm =  cont as? ThisSM ?: object : ThisSM {
        fun resume(…) {
            postItem(null, this)
        }
    }
    switch (sm.label) {
        case 0:
            sm.item = item
            sm.label = 1
            requestToken(sm)
        case 1:
            val item = sm.item
            val token = sm.result as Token
            sm.label = 2
            createPost(token, item, sm)
        …
    }
```

可以看到，这里 kotlin 实际上是利用状态机来实现 CPS 风格的，对于每一个 suspend point，都有一个唯一的状态值对应，状态机根据不同的状态，执行不同的过程

总结一下:

1. 一个 suspend 函数定义了一个状态机
2. 一个协程就是一个状态机的实例，这个实例就是将其调用的 suspend 函数组合在了一起
