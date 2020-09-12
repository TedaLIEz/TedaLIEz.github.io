---
title: Jetpack Compose简介与思考
date: 2020-07-25 09:22:01
tags: android, jetpack compose
---

Jetpack Compose可以认为是Android对UI代码架构的演进，核心目的是为了让安卓自身的UI代码能够跟随现代UI开发的步伐


## 现代UI开发的特性


现代UI开发的一个重要特点在于其代码为声明式的代码结构，从数学关系上来看，UI的代码构成可以描述为对某一时刻状态的一个函数，即`UI=F(n)`，其中n表示了当前的交互状态。参考MVVM的设计来说，这个状态可以由ViewModel来管理

## 安卓的UI开发现状


安卓的UI开发思路和代码结构已经明显老化，主要体现在下面一些方面:

1. 仍旧有xml这样的配置文件，即使出现了ViewBinding也无济于事
2. 构建UI时有大量的命令式代码，这种代码散落在Activity/Fragment中，不利于维护


## 命令式编程与声明式编程的争论

命令式编程风格的问题在于程序非常依赖过程，换言之，如果代码编写的顺序出错，那么程序也会出错；而声明式编程则是只关心结果，编写代码时，只需要把开发者最终想要的结果直接写入代码，如果需要变化，则直接修改你的声明代码。并且声明式风格的UI代码在状态改变的任意时刻都是**同一套代码**构建UI, 这有助于提升代码质量

声明式风格编码特点在于:

    Describe your UI right now. For any value of now.

这种风格是一种状态无关代码(status independent)

## Jetpack Compose

Jetpack Compose的目的就在于改变现有的UI开发思路，完全向声明式函数演进，同时提供了一些便捷功能，例如一个常规的UI声明如下:


```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        findViewById<TextView>(R.id.tv).text = "Android"
    }
}
```

对于复杂的UI界面，这种UI代码的编码方式是可以极其零散的, 可能一部分写在xml里面，一部分又写在代码里，同时代码里的编码可以任意地乱序（命令式代码的弊端）。而Compose出现后，我们的编码方式就可以变化为



```kotlin

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyApp {
                Greeting(name = "Android")
            }
        }
    }
}


@Composable
fun MyApp(content: @Composable() () -> Unit) {
    MaterialTheme {
        content()
    }
}

@Composable
fun Greeting(name: String) {
    Surface(color = Color.Yellow) {
        Text(text = "Hello $name!", modifier = Modifier.padding(24.dp))
    }
}


```

乍一看这种编码风格非常像Flutter，React，Vue的UI代码风格，本质上这些前端框架的UI设计都遵循了声明式函数风格，通过将编码方式转化为声明式函数风格后，我们间接地获得了以下收益

1. 各个函数独立，复用性提升
2. UI代码紧凑

复用性提升的直接效果之一就是我们可以在IDE里面直接preview UI效果，这个也是Android Studio 4.2后内置的能力，只需要给你的`@Composable`函数加上一个`@Preview`标签并进行编译，这个函数自己的UI效果就能直接在IDE上展示

UI代码紧凑则明确了任何对UI的修改都有唯一的入口，例如在上面的例子中，`Activity`中一定是`setContent`




### Jetpack Compose的状态管理

声明式风格的UI都面临一个问题就是如何处理状态改变，Jetpack Compose和其他常见的声明式UI框架一致，采用了state这个概念，通过声明一个state对象，我们根据这个state对象构造自己的UI和其子View，当state改变时，动态的改变这个UI的声明，例如:


```kotlin
@Composable
fun MyScreenContent() {
    val counterState = state { 0 }
    Counter(
        count = counterState.value,
        updateCount = { newCount ->
            counterState.value = newCount
        }
    )
}


@Composable
fun Counter(count: Int, updateCount: (Int) -> Unit) {
    Button(onClick = { updateCount(count+1) }) {
        Text("I've been clicked $count times")
    }
}

```

这个例子中，我们就能发现`MyScreenContent`中内含了一个counterState对象，它通过counterState构建了一个Counter，而Counter也是counterState的一个函数，当counterState变化时，Counter也会重新构建，UI因此也发生变化


### Jetpack Compose中Lifecycle

Android开发非常强调Lifecycle-aware，compose也是如此。为此Jetpack Compose提出了Efforts这个概念，在这个概念中提供了onCommit, onPreCommit, onActive, onDispose一系列函数，用来监听生命周期变化


## 感想和思考

似乎在UI开发上，声明式编程已经成为了主流的方向。随着SwiftUI，Flutter，Jetpack Compose的出现，越来越多的新兴UI开发框架抛弃了现有的MVC，MVP模式，而走向声明式UI方向，随着声明式UI被引入移动端开发，新兴的MVI模式也进入了大众视野，并逐渐被人们接受
