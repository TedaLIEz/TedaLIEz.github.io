---
title: Kotlin协程实战
date: 2019-12-07 19:49:40
tags: kotlin
---

- [学习路径](#%e5%ad%a6%e4%b9%a0%e8%b7%af%e5%be%84)
- [协程尝试解决什么问题](#%e5%8d%8f%e7%a8%8b%e5%b0%9d%e8%af%95%e8%a7%a3%e5%86%b3%e4%bb%80%e4%b9%88%e9%97%ae%e9%a2%98)
- [如何理解 suspend 关键字](#%e5%a6%82%e4%bd%95%e7%90%86%e8%a7%a3-suspend-%e5%85%b3%e9%94%ae%e5%ad%97)
- [协程中的异常处理](#%e5%8d%8f%e7%a8%8b%e4%b8%ad%e7%9a%84%e5%bc%82%e5%b8%b8%e5%a4%84%e7%90%86)
- [最佳实践](#%e6%9c%80%e4%bd%b3%e5%ae%9e%e8%b7%b5)

## 学习路径

1. https://codelabs.developers.google.com/codelabs/kotlin-coroutines/#0
2. 官方文档
3. https://kaixue.io/kotlin-coroutines-1/

下面说下个人理解

## 协程尝试解决什么问题

从编码角度，协程尝试解决了回调地狱的问题
从实现上，协程尝试减少了线程切换的开销

## 如何理解 suspend 关键字

TL;DR:

> 使用 suspend 包装一个耗时操作，同时将耗时操作用 withContext()包装起来，并指定其要使用的线程

suspend 关键字在 kotlin 中用于函数声明，这里需要做一个说明：

    一个suspend的函数并不代表其不会在主线程上执行

在 Dispatchers.MAIN 的上下文下，你的函数就是会在主线程上执行的

另外一条说明：

    suspend关键字并不是真正实现挂起

它只是一个提醒，提醒调用者我是一个耗时函数，我被我的创建者用挂起的方式放到后台执行，你需要用协程来调用我

例如:

```Kotlin

    fun test() : String {
        viewModelScope.launch {
            Log.d(TAG, "test in IO thread: " + Thread.currentThread())
            delay(1_000)
            Log.d(TAG, "do we really sleep about 1000ms? " + Thread.currentThread())
        }
        return "Test"
    }

    fun onMainViewClicked() {

        viewModelScope.launch {
            val rst = test()
            _snackBar.postValue(rst)
        }
    }
```

我们说，test()在 viewModelScope.launch 时启动了一个协程，这个线程 delay 了 1000ms 后返回 test，那么这段代码的表现就是

```
2019-12-02 21:21:39.615 21445-21445/com.example.android.kotlincoroutines D/MainViewModel: test in IO thread: Thread[main,5,main]
2019-12-02 21:21:40.620 21445-21445/com.example.android.kotlincoroutines D/MainViewModel: do we really sleep about 1000ms? Thread[main,5,main]
```

在第一条日志出现时，\_snackBar 就已经调用了 postValue 了，delay 函数并没有阻塞住我们的 viewModelScope

如果我们希望延时 1s 后 postValue 呢？下面这个写法 ok 吗？

```Kotlin
    fun test() : String {
        viewModelScope.async {
            Log.d(TAG, "test in IO thread: " + Thread.currentThread())
            delay(1_000)
            Log.d(TAG, "do we really sleep about 1000ms? " + Thread.currentThread())
        }
        return "Test"
    }
    /**
     * Wait one second then display a snackbar.
     */
    fun onMainViewClicked() {

        viewModelScope.launch {
            val rst = test()
            _snackBar.postValue(rst)
        }
    }

```

结果和 launch 一致，原因也很简单，因为我们没有对 async 内的协程做任何操作，只是让它执行.

为了让它等待 1s，就有两种方式

1. async#await()

```kotlin
suspend fun test() : String {
    viewModelScope.async {
        Log.d(TAG, "test in IO thread: " + Thread.currentThread())
        delay(1_000)
        Log.d(TAG, "do we really sleep about 1000ms? " + Thread.currentThread())
    }.await()
    return "Test"
}

/**
* Wait one second then display a snackbar.
*/
fun onMainViewClicked() {

    viewModelScope.launch {
        val rst = test()
        _snackBar.postValue(rst)
    }
}

```

我们会发现一旦调用了 await()，编译器就会要求我们声明 test 为 suspend 函数，原因很简单。如上文所述，await()是被声明为 suspend，它被声明为耗时函数，那么你需要放在一个 suspend 函数去调用

结果:

```
2019-12-02 21:26:10.198 21809-21809/com.example.android.kotlincoroutines D/MainViewModel: test in IO thread: Thread[main,5,main]
2019-12-02 21:26:11.202 21809-21809/com.example.android.kotlincoroutines D/MainViewModel: do we really sleep about 1000ms? Thread[main,5,main]
```

postValue 将会在第二条日志打印时调用，满足延时效果

2. withContext()

```kotlin
    suspend fun test() : String {
        withContext(viewModelScope.coroutineContext) {
            Log.d(TAG, "test in IO thread: " + Thread.currentThread())
            delay(1_000)
            Log.d(TAG, "do we really sleep about 1000ms? " + Thread.currentThread())
        }
        return "Test"

    }
    /**
     * Wait one second then display a snackbar.
     */
    fun onMainViewClicked() {

        viewModelScope.launch {
            val rst = test()
            _snackBar.postValue(rst)
        }
    }
```

效果同 1，但这里可以额外说一下，withContext 允许带返回值，你可以理解就是 async#await()的整合版。

这里需要对 1 和 2 做一个总结，async 配合 await()和 withContext 到底干了什么？简单来说

    就是帮助你将线程切走，同时当传入的lambda有返回值时，帮你自动切回来

啥意思呢？比如你有下面一段代码:

```kotlin

    suspend fun getUser() : User{
        return withContext(Dispatchers.IO) {
            return network.fetchUser()   // IO Thread
        }
    }


    fun showUser() {
        viewModelScope.launch {
            val user = getUser()   // Main Thread
            _user.postValue(user)
        }
    }
```

我们认为`network.fetchUser()`是一个耗时操作，那么我们用 withContext(Dispatchers.IO)包裹起来，丢给 IO 线程处理，当 network.fetchUser()返回时，withContext 自动将线程切回 MAIN 线程，回到 showUser()的 user 变量处，这里也是 MAIN 线程

## 协程中的异常处理

协程中的异常处理取决于你创建协程的方式

如果是 launch 创建的协程，异常会立即抛出，同时会导致其树状关系的所有 job 失败，因此你需要在协程内代码自行使用 try-catch

例如:

```kotlin
val job: Job = Job()
val scope = CoroutineScope(Dispatchers.Default + job)
// may throw Exception
fun doWork(): Deferred<String> = scope.async { ... }   // (1)
fun loadData() = scope.launch {
    try {
        doWork().await()                               // (2)
    } catch (e: Exception) { ... }
}
```

这个时候(2)中捕获的异常也会立即使其父 job 失败，一种解决方案就是用 SupervisorJob 来作为父 job

    A failure or cancellation of a child does not cause the supervisor job to fail and does not affect its other children.

这么做能捕获的前提是你的 async 也在 SupervisorJob 构造的 scope 中执行，不然也是会 crash 的, 例如:

```kotlin
val job = SupervisorJob()
val scope = CoroutineScope(Dispatchers.Default + job)

fun loadData() = scope.launch {
    try {
        async {                                         // (1)
            // may throw Exception, still crash
        }.await()
    } catch (e: Exception) { ... }
}
```

另一种解决方案就是使用 coroutineScope 函数包裹你的 async 调用

```kotlin
val job = SupervisorJob()
val scope = CoroutineScope(Dispatchers.Default + job)

// may throw Exception
suspend fun doWork(): String = coroutineScope {     // (1)
    async { ... }.await()
}

fun loadData() = scope.launch {                       // (2)
    try {
        doWork()
    } catch (e: Exception) { ... }
}
```

如果是 async 创建的协程，异常会在你调用 await 的位置抛出，如果没有调用 await，不会有抛出异常的可能，因此你需要在 await 调用的位置使用 try-catch

如果是 withContext 创建的协程，同 launch 的处理

最后，你都可以创建一个 CoroutineExceptionHandler 对象传入到你的 scope 中来统一处理异常

```kotlin

private val coroutineExceptionHandler: CoroutineExceptionHandler =
    CoroutineExceptionHandler { _, throwable ->
      //2
      coroutineScope.launch(Dispatchers.Main) {
        //3
        errorMessage.visibility = View.VISIBLE
        errorMessage.text = getString(R.string.error_message)
      }

      GlobalScope.launch { println("Caught $throwable") }
    }

private val coroutineScope =
    CoroutineScope(Dispatchers.Main + parentJob + coroutineExceptionHandler)
```

## 最佳实践

1. 实现 suspend 函数时，保证其线程安全，**可以在任何线程(Dispatcher)调用**

```kotlin
suspend fun login() : Result {
    view.showLoading()

    val result = withContext(Dispatchers.IO) {
        loginBlockingCall()
    }

    view.hideLoading()
    return result
}
```

上面的 login 有潜在的 crash 风险，调用方极有可能将这个函数放在 non-Main dispatcher 中调用，为了避免这种类型的 crash，请将你的函数设计成线程安全的

```kotlin
suspend fun login() : Result = withContext(Dispachers.Main) {
        view.showLoading()

        val result = withContext(Dispatchers.IO) {
            loginBlockingCall()
        }

        view.hideLoading()
        return result
    }
```

2. Android 开发应避免 GlobalScope
   因为 GlobalScope 是跟随 Application 的生命周期，使用 GlobalScope 下的协程会带来资源泄漏的风险
