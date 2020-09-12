---
title: Kotlin协程异常处理
date: 2020-09-12 09:09:48
tags: kotlin, 协程
---

Kotlin 的协程满足结构化语义：

1. A parent-Coroutine finishes only after all its child-Coroutines have finished.

2. When a parent-Coroutine or scope finishes abnormally, either through cancelation or through an exception, all its child-Coroutines, that are still active, are canceled and new child-Coroutines can no longer be launched.

3. When a child-Coroutine finishes abnormally, its parent-Coroutine or scope finishes abnormally.

针对第三点，无论是`CoroutineScope.async`或`CoroutineScope.launch`，以异常结束，只要是在一个 scope 中，他的父协程也会以异常结束。`async`和`launch`唯一不同的地方在于`async`需要调用 await 才会执行协程内容。即使我们对子协程进行了 try-catch 处理异常，父协程仍旧会拿着这个异常作为结果结束，例如下面这个代码

```kotlin
val deferredResult = async {
    // This child-Coroutine throws an exception
    throw SomeSillyException()
}
textView.rating = try {
    deferredResult.await()
} catch (e: SomeSillyException) {
    // We catch the exception and assign a default value
    -1
}
// At this point, this parent-Coroutine has finished with
// 'SomeSillyException' as well, even though we caught
// the exception just now!

```

上面的代码中我们可以看到，即使我们对 await 进行了`try-catch`, await 抛出的异常仍旧会走到父协程中抛出（满足`父协程也会以异常结束`这一原则.

```kotlin

viewModelScope.launch {
    coroutineScope {
        val deferred= async {
            throw IllegalArgumentException()
        }

        try {
            deferred.await()
        } catch (e: Exception) {
            // 我们在coroutineScope内catch了异常，但是根据协程机制，这个异常仍旧会抛给viewModelScope，因此还是crash
        }
    }
}

```

对上面的代码也可以这么理解，对 await()的`try-catch`是对 deferred 的结果进行了异常捕获，但对于 deferred 协程自身的异常，是没有处理的。下面这个代码则是直接对 deferred 的协程进行了异常处理

```kotlin
viewModelScope.launch {
    try {
        coroutineScope {
            val deferred= async {
                throw IllegalArgumentException()
            }
            deferred.await()
        }
    } catch (e: Exception) {
        // 我们在coroutineScope外catch异常，此时await不会导致crash
    }
}

```

那么如何能够让子协程抛出异常的情况下，父协程不会终止所以其子协程和自己呢？这就引入了 supervisorScope

```kotlin
val result = supervisorScope {
    ...
    val supervisedChild1 = this.launch { ... }
    ...
    val supervisedChild2 = this.async { ... }
    ...
}
```

此时，若 child1 或 child2 任意一个抛出异常，也不会使另一个 child 和`supervising parent`停止

最终我们得到了协程的 Structured Concurrency 含义:

1. A parent-Coroutine finishes only after all its child-Coroutines have finished.

2. When a parent-Coroutine or scope finishes abnormally, either through cancelation or through an exception, all its child-Coroutines, that are still active, are canceled and new child-Coroutines can no longer be launched.

3. When a child-Coroutine finishes abnormally, its parent-Coroutine or scope (a) finishes abnormally if the parent is not a supervisor or (b) keeps running if the parent is a supervisor.

ref: [https://medium.com/@elizarov/structured-concurrency-722d765aa952](https://medium.com/@elizarov/structured-concurrency-722d765aa952)

ref: [https://medium.com/the-kotlin-chronicle/coroutine-exceptions-supervision-15056802998b](https://medium.com/the-kotlin-chronicle/coroutine-exceptions-supervision-15056802998b)

ref: [https://medium.com/the-kotlin-chronicle/coroutine-exceptions-3378f51a7d33](https://medium.com/the-kotlin-chronicle/coroutine-exceptions-3378f51a7d33)
