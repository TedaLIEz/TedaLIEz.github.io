---
title: v8执行WebAssembly遇到的问题和解决方法
date: 2019-06-06 19:20:44
tags: v8, WebAssembly
---

- [问题的简单分析](#%e9%97%ae%e9%a2%98%e7%9a%84%e7%ae%80%e5%8d%95%e5%88%86%e6%9e%90)

  - [查阅`d8.cc`](#%e6%9f%a5%e9%98%85d8cc)
    - [SourceGroup::Execute 简单分析](#sourcegroupexecute%e7%ae%80%e5%8d%95%e5%88%86%e6%9e%90)
    - [CompleteMessageLoop 分析](#completemessageloop%e5%88%86%e6%9e%90)
  - [解决方案](#%e8%a7%a3%e5%86%b3%e6%96%b9%e6%a1%88)

最近在 7.2 分支上做一些 WebAssembly 能力的验证, 但是发现了一些奇怪的现象, 具体的问题描述可以参考[https://stackoverflow.com/questions/56428990/webassembly-instantiate-didnt-call-then-nor-catch-in-v8-embedded](https://stackoverflow.com/questions/56428990/webassembly-instantiate-didnt-call-then-nor-catch-in-v8-embedded)

简单来说,

```javascript
WebAssembly.instantiate(
  new Uint8Array([
    0,
    97,
    115,
    109,
    1,
    0,
    0,
    0,
    1,
    8,
    2,
    96,
    1,
    127,
    0,
    96,
    0,
    0,
    2,
    8,
    1,
    2,
    106,
    115,
    1,
    95,
    0,
    0,
    3,
    2,
    1,
    1,
    8,
    1,
    1,
    10,
    9,
    1,
    7,
    0,
    65,
    185,
    10,
    16,
    0,
    11
  ]),
  {
    js: {
      _: console.log("Called from WebAssembly Hello world")
    }
  }
)
  .then(function(obj) {
    console.log("Called with instance " + obj);
  })
  .catch(function(err) {
    console.log("Called with error " + err);
  });
```

上面这段代码既没有执行 then 函数, 又没有执行 catch 函数, 明显不符合 Web API 的预期.

## 问题的简单分析

为了分析这个问题, 我尝试在我自己编译的 d8 中执行上面这段脚本, d8 是 v8 编译时默认生成的一个可交互 JS 运行环境, 简单理解就是一个 JS 的 REPL
{% asset_img v8_WASM_1.jpg %}

很神奇, 我居然之后的 REPL 中获得了 catch 的调用, 也就是说 d8 一次执行结束后也没有返回 Promise 的 resolve 或者 reject. 那么这里就需要查查 d8 实现的方式了

### 查阅`d8.cc`

直接来到`src/d8.cc`, 来到最关键的函数`Shell::Main`

```c++
int Shell::RunMain(Isolate* isolate, int argc, char* argv[], bool last_run) {
  for (int i = 1; i < options.num_isolates; ++i) {
    options.isolate_sources[i].StartExecuteInThread();
  }
  {
    SetWaitUntilDone(isolate, false);
    if (options.lcov_file) {
      debug::Coverage::SelectMode(isolate, debug::Coverage::kBlockCount);
    }
    HandleScope scope(isolate);
    Local<Context> context = CreateEvaluationContext(isolate);
    bool use_existing_context = last_run && use_interactive_shell();
    if (use_existing_context) {
      // Keep using the same context in the interactive shell.
      evaluation_context_.Reset(isolate, context);
    }
    {
      Context::Scope cscope(context);
      InspectorClient inspector_client(context, options.enable_inspector);
      PerIsolateData::RealmScope realm_scope(PerIsolateData::Get(isolate));
      options.isolate_sources[0].Execute(isolate);  // 关键
      CompleteMessageLoop(isolate);   // 关键
    }
    if (!use_existing_context) {
      DisposeModuleEmbedderData(context);
    }
    WriteLcovData(isolate, options.lcov_file);
  }
  CollectGarbage(isolate);
  for (int i = 1; i < options.num_isolates; ++i) {
    if (last_run) {
      options.isolate_sources[i].JoinThread();
    } else {
      options.isolate_sources[i].WaitForThread();
    }
  }
  CleanupWorkers();
  return 0;
}
```

关键看这两行:

```c++
    options.isolate_sources[0].Execute(isolate);  // 关键
    CompleteMessageLoop(isolate);   // 关键
```

options.isolate_sources[0]实际上是一个 SourceGroup 对象

#### SourceGroup::Execute 简单分析

```c++
void SourceGroup::Execute(Isolate* isolate) {
  bool exception_was_thrown = false;
  for (int i = begin_offset_; i < end_offset_; ++i) {
    // 参数检查, 这里略去

    // Use all other arguments as names of files to load and run.
    HandleScope handle_scope(isolate);
    Local<String> file_name =
        String::NewFromUtf8(isolate, arg, NewStringType::kNormal)
            .ToLocalChecked();
    Local<String> source = ReadFile(isolate, arg);
    if (source.IsEmpty()) {
      printf("Error reading '%s'\n", arg);
      base::OS::ExitProcess(1);
    }
    Shell::set_script_executed();
    if (!Shell::ExecuteString(isolate, source, file_name, Shell::kNoPrintResult,
                              Shell::kReportExceptions,
                              Shell::kProcessMessageQueue)) {  // 关键
      exception_was_thrown = true;
      break;
    }
  }
  if (exception_was_thrown != Shell::options.expected_to_throw) {
    base::OS::ExitProcess(1);
  }
}
```

再进去看`Shell::ExecuteString`

```c++
bool Shell::ExecuteString(Isolate* isolate, Local<String> source,
                          Local<Value> name, PrintResult print_result,
                          ReportExceptions report_exceptions,
                          ProcessMessageQueue process_message_queue) {
  HandleScope handle_scope(isolate);
  TryCatch try_catch(isolate);
  try_catch.SetVerbose(true);

  MaybeLocal<Value> maybe_result;
  bool success = true;
  {
    PerIsolateData* data = PerIsolateData::Get(isolate);
    Local<Context> realm =
        Local<Context>::New(isolate, data->realms_[data->realm_current_]);
    Context::Scope context_scope(realm);
    MaybeLocal<Script> maybe_script;
    Local<Context> context(isolate->GetCurrentContext());
    ScriptOrigin origin(name);

    //  部分代码略去

    Local<Script> script;
    if (!maybe_script.ToLocal(&script)) {
      // Print errors that happened during compilation.
      if (report_exceptions) ReportException(isolate, &try_catch);
      return false;
    }

    // 部分代码略去
    maybe_result = script->Run(realm);
    if (options.code_cache_options ==
        ShellOptions::CodeCacheOptions::kProduceCacheAfterExecute) {
      // Serialize and store it in memory for the next execution.
      ScriptCompiler::CachedData* cached_data =
          ScriptCompiler::CreateCodeCache(script->GetUnboundScript());
      StoreInCodeCache(isolate, source, cached_data);
      delete cached_data;
    }
    if (process_message_queue && !EmptyMessageQueues(isolate)) success = false;  // 关键
    data->realm_current_ = data->realm_switch_;
  }
  Local<Value> result;
  if (!maybe_result.ToLocal(&result)) {
    DCHECK(try_catch.HasCaught());
    // Print errors that happened during execution.
    if (report_exceptions) ReportException(isolate, &try_catch);
    return false;
  }
  DCHECK(!try_catch.HasCaught());
  if (print_result) {
    if (options.test_shell) {
      if (!result->IsUndefined()) {
        // If all went well and the result wasn't undefined then print
        // the returned value.
        v8::String::Utf8Value str(isolate, result);
        fwrite(*str, sizeof(**str), str.length(), stdout);
        printf("\n");
      }
    } else {
      v8::String::Utf8Value str(isolate, Stringify(isolate, result));
      fwrite(*str, sizeof(**str), str.length(), stdout);
      printf("\n");
    }
  }
  return success;
}
```

看着就跟官方 demo 里面执行 hello world 差不多, 但是其中的一行有点令人在意.

```c++
if (process_message_queue && !EmptyMessageQueues(isolate)) success = false;
```

这个 EmptyMessageQueues 是什么意思?

```c++
bool Shell::EmptyMessageQueues(Isolate* isolate) {
  return ProcessMessages(
      isolate, []() { return platform::MessageLoopBehavior::kDoNotWait; });
}

bool ProcessMessages(
    Isolate* isolate,
    const std::function<platform::MessageLoopBehavior()>& behavior) {
  while (true) {
    i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
    i::SaveContext saved_context(i_isolate);
    i_isolate->set_context(i::Context());
    SealHandleScope shs(isolate);
    while (v8::platform::PumpMessageLoop(g_default_platform, isolate,
                                         behavior())) {
      isolate->RunMicrotasks();
    }
    if (g_default_platform->IdleTasksEnabled(isolate)) {
      v8::platform::RunIdleTasks(g_default_platform, isolate,
                                 50.0 / base::Time::kMillisecondsPerSecond);
    }
    HandleScope handle_scope(isolate);
    PerIsolateData* data = PerIsolateData::Get(isolate);
    Local<Function> callback;
    if (!data->GetTimeoutCallback().ToLocal(&callback)) break;
    Local<Context> context;
    if (!data->GetTimeoutContext().ToLocal(&context)) break;
    TryCatch try_catch(isolate);
    try_catch.SetVerbose(true);
    Context::Scope context_scope(context);
    if (callback->Call(context, Undefined(isolate), 0, nullptr).IsEmpty()) {
      Shell::ReportException(isolate, &try_catch);
      return false;
    }
  }
  return true;
}
```

这里的 ProcessMessages 就很有意思了.

首先是一个死循环, 接着在这个死循环里面我们有另外一个 while 循环

```c++
while (v8::platform::PumpMessageLoop(g_default_platform, isolate,
                                        behavior())) {
    isolate->RunMicrotasks();
}
```

这里`behavior`是外部传入的 lambda, 表示 PumpMessageLoop 的等待参数, 我们看一下这个参数的含义

```c++

/**
 * Pumps the message loop for the given isolate.
 *
 * The caller has to make sure that this is called from the right thread.
 * Returns true if a task was executed, and false otherwise. Unless requested
 * through the |behavior| parameter, this call does not block if no task is
 * pending. The |platform| has to be created using |NewDefaultPlatform|.
 */
V8_PLATFORM_EXPORT bool PumpMessageLoop(
    v8::Platform* platform, v8::Isolate* isolate,
    MessageLoopBehavior behavior = MessageLoopBehavior::kDoNotWait);

enum class MessageLoopBehavior : bool {
  kDoNotWait = false,
  kWaitForWork = true
};

/**
* Runs the Microtask Work Queue until empty
* Any exceptions thrown by microtask callbacks are swallowed.
*/
void RunMicrotasks();
```

根据定义, kDoNotWait 为默认属性, 表示不等待消息到达, 即表示 PumpMessageLoop 如果没有任务, 不会阻塞线程. 返回 true 表示有任务被执行, false 表示没有.

刚才的 EmptyMessageQueues 函数中直接返回了 kDoNotWait, 也就是说只要没有消息, 就直接返回 false, 继续执行代码; 如果为 true, 则将微队列任务全部执行.

```c++
if (g_default_platform->IdleTasksEnabled(isolate)) {
    v8::platform::RunIdleTasks(g_default_platform, isolate,
                                50.0 / base::Time::kMillisecondsPerSecond);
}
HandleScope handle_scope(isolate);
PerIsolateData* data = PerIsolateData::Get(isolate);
Local<Function> callback;
if (!data->GetTimeoutCallback().ToLocal(&callback)) break;
Local<Context> context;
if (!data->GetTimeoutContext().ToLocal(&context)) break;
TryCatch try_catch(isolate);
try_catch.SetVerbose(true);
Context::Scope context_scope(context);
if (callback->Call(context, Undefined(isolate), 0, nullptr).IsEmpty()) {
    Shell::ReportException(isolate, &try_catch);
    return false;
}
```

默认不开启 IdleTasksEnabled, 这里暂时跳过. 后面这段代码结合 d8 源码来看, 其目的就是尝试去获取 Isolate 中是否还有之前通过 timeout 函数设置的 callback, 如果有的话就尝试执行, 如果已经没有则 break 掉 while(true)循环

总结一下, 在 ExecuteString 函数中, 会以 kDoNotWait 方式去清空微队列中的任务, 并且执行所有的 timeout 回调

#### CompleteMessageLoop 分析

RunMain 中的第二行关键调用就是 CompleteMessageLoop

```c++
void Shell::CompleteMessageLoop(Isolate* isolate) {
  auto get_waiting_behaviour = [isolate]() {
    base::MutexGuard guard(isolate_status_lock_.Pointer());
    DCHECK_GT(isolate_status_.count(isolate), 0);
    i::Isolate* i_isolate = reinterpret_cast<i::Isolate*>(isolate);
    i::wasm::WasmEngine* wasm_engine = i_isolate->wasm_engine();
    bool should_wait = (options.wait_for_wasm &&
                        wasm_engine->HasRunningCompileJob(i_isolate)) ||
                       isolate_status_[isolate];
    return should_wait ? platform::MessageLoopBehavior::kWaitForWork
                       : platform::MessageLoopBehavior::kDoNotWait;
  };
  ProcessMessages(isolate, get_waiting_behaviour);
}
```

这里居然直接写了一段跟 WASM 强相关的逻辑:

```c++
bool should_wait = (options.wait_for_wasm &&
                        wasm_engine->HasRunningCompileJob(i_isolate)) ||
                       isolate_status_[isolate];
```

`wait_for_wasm`默认为`true`, 也就是说, 只要 HasRunningCompileJob 也为 true, 我们投递给 PumpMessageLoop 的 MessageLoopBehavior 即 kWaitForWork.

当 MessageLoopBehavior 为 kWaitForWork 时, v8 会等待任务执行结束, 没有任务则会直接阻塞线程.

虽然还没有具体分析, 但目前可以推测 WASM 在 7.2 这个版本编译的实现变成了一个异步任务, 而外部如果想要知道这个执行结束的消息一定会依赖 Promise#resolve 方式来执行, d8 采用了 kWaitForWork 的方式来确保线程能够等待到 WASM 编译结束, 调用其对应的 resolve 或 reject 函数. (这里还需要再具体地根据源码分析, 如果以后被打脸了就再修改这里的分析内容)

### 解决方案

d8 中对 wasm 的处理方式对我们是否有所启发呢？
对比一下我和 d8 的代码, 很快就可以得出结论了.

在我的代码中

```c++
// sample wasm javascript code here.
  const char *csource = R"(
    WebAssembly.instantiate(new Uint8Array([0,97,115,109,1,0,0,0,1,8,2,96,1,127,0,96,0,0,2,8,1,2,106,
      115,1,95,0,0,3,2,1,1,8,1,1,10,9,1,7,0,65,185,10,16,0,11]),
      {js:{_:console.log('Called from WebAssembly Hello world')}}).then(function(obj) {
        log('Called with instance ' + obj);
      }).catch(function(err) {
        log('Called with error ' + err);
      });
  )"; // should call my Hello World log and trigger the error or return the instance successfully

  v8::HandleScope handle_scope(isolate);
  auto ctx = persistentContext.Get(isolate);
  v8::Context::Scope context_scope(ctx);
  v8::TryCatch try_catch(isolate);
  v8::Local<v8::String> source = v8::String::NewFromUtf8(isolate, csource,
                                                         v8::NewStringType::kNormal).ToLocalChecked();

  v8::Local<v8::Script> script =
      v8::Script::Compile(ctx, source).ToLocalChecked();
  v8::Local<v8::Value> result;
  if (!script->Run(ctx).ToLocal(&result)) {
    ReportException(isolate, &try_catch); // report exception, ignore the implementation here
    return;
  }
  // Convert the result to an UTF8 string and print it.
  v8::String::Utf8Value utf8(isolate, result);
  __android_log_print(ANDROID_LOG_INFO, "V8Native", "%s\n", *utf8);
```

当执行这段代码时, `script->Run(ctx)`可以理解为 REPL 中的 Evaluate 环节,
`__android_log_print(ANDROID_LOG_INFO, "V8Native", "%s\n", *utf8)`可以理解为 REPL 中的 Print 环节.

但这里我们少了个 Loop, 这里引入了问题. WebAssembly.instantiate 的异步编译导致我们不再有机会执行异步编译成功/失败后的 resolve 或者 reject 方法, 我们没有等待 WebAssembly 执行结束就直接返回了.

那么如何解决这个问题呢?

最暴力的方案就是直接在我们的函数结尾加上一段:

```c++
while (v8::platform::PumpMessageLoop(platform, isolate, v8::platform::MessageLoopBehavior::kWaitForWork)) {
    isolate->RunMicrotasks();
}
```

这样我们会强制要求等待所有的消息到达并全部执行后才会结束, 这样解决了我的 demo 问题, 但这个方式显然不解决通用场景.

`v8::platform::MessageLoopBehavior::kWaitForWork`表示如果没有消息了, 那么 v8 会选择阻塞线程来等待, 你可以类比这个为 epoll_wait(), 那么我们如果有一段 js 代码, 例如:

```javascript
console.log("Hello world");
```

这段代码本身不涉及任何微队列或异步问题, Run 结束后就应该立刻返回, 但是 kWaitForWork 会告诉 v8, 没有消息也必须等待, 那么我们的函数也就永远不会返回了.

实际上比较合理的做法应该是在你的 JS 执行线程中进行一个固定的 timer 操作, 类似 node.js 中的 Event Loop 去固定触发一个定时器, 这个定时器一方面要及时去触发 setTimeout 或 setInterval 之类的定时操作, 另一方面则是需要让 v8 及时地去刷新自己的消息队列并执行微队列任务, 例如:

```c++
void eventLoop() {
    while (v8::platform::PumpMessageLoop(platform, isolate)) {
        isolate->RunMicrotasks();
    }
    m_spTimer->updateCallback();
}
```

这样就解决了我的问题, 我的 WASM 也成功地在 Android 设备上执行了.这篇文章有一些自己的主观臆断, 如果有与我分析不一致的情况, 欢迎通过邮件或 github issues 的方式讨论.
