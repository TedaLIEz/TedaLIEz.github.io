---
title: Android自定义View中一些参数的含义
date: 2019-11-16 20:31:21
tags: android
---

- [Android 自定义 View 中一些参数的含义](#android-%e8%87%aa%e5%ae%9a%e4%b9%89-view-%e4%b8%ad%e4%b8%80%e4%ba%9b%e5%8f%82%e6%95%b0%e7%9a%84%e5%90%ab%e4%b9%89)
  - [TL;DR](#tldr)
    - [AttributeSet](#attributeset)
    - [defStyleAttr](#defstyleattr)
    - [defStyleRes](#defstyleres)
  - [小结](#%e5%b0%8f%e7%bb%93)
  - [P.S 为什么不使用集联方式实现自定义 View](#ps-%e4%b8%ba%e4%bb%80%e4%b9%88%e4%b8%8d%e4%bd%bf%e7%94%a8%e9%9b%86%e8%81%94%e6%96%b9%e5%bc%8f%e5%ae%9e%e7%8e%b0%e8%87%aa%e5%ae%9a%e4%b9%89-view)
  - [References](#references)

# Android 自定义 View 中一些参数的含义

## TL;DR

不要理会`View(Context, AttributeSet, defStyleAttr)`和`View(Context, AttributeSet, defStyleAttr, defStyleRes)`, 关心`View(Context)`, `View(Context, AttributeSet)`, 范例代码:

```java
class MyView extends SomeView {
    public MyView(Context context) {
        super(context);
        init(context, null, 0);
    }

    public MyView(Context context, AttributeSet attrs) {
        super(context,attrs);
        init(context, attrs, 0);
    }

    public MyView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init(context, attrs, defStyle);
    }

    private void init(Context context, AttributeSet attrs, int defStyle) {
        // do something cool
    }
}
```

### AttributeSet

这个参数代表了所有在 XML 中申明的属性, 例如:

```java
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@string/hello"
    undeclaredAttr="value for undeclared attr" />

```

所有 XML 中调用都会涉及 View 的两个参数的构造函数, 此时 AttributeSet 就会包含四个属性: `layout_width`, `layout_height`, `text`, `undeclaredAttr`

可能有的人会好奇这些属性是从哪里来的，实际上在自定义 View 时，我们可以在 attrs 中声明`declare-styleable`来为我们的 View 增加属性, 例如:

```xml
<declare-styleable name="ImageView">
  <!-- Sets a drawable as the content of this ImageView. -->
  <attr name="src" format="reference|color" />

  <!-- ...snipped for brevity... -->

</declare-styleable>
```

上面这个就是 ImageView 中 src 属性的由来, 有了这个定义, Android 的 ImageView 才能通过 xml 来设置 src 属性

通常在`View(Context context, AttributeSet attrs)`, 我们会使用`Theme.obtainStyledAttributes(android.util.AttributeSet, int[], int, int)`来获取 AttributeSet 中的属性, 主要原因是因为我们往往需要依赖一套机制来解决在 xml 配置时使用的各种关系

> For example, if you define style=@style/MyStyle in your XML, this method resolves MyStyle and adds its attributes to the mix. In the end, obtainStyledAttributes() returns a TypedArray which you can use to access the attributes.

### defStyleAttr

View 的构造函数有一种方法签名: `View(Context context, AttributeSet attrs, int defStyleAttr)`, 其中的 defStyleAttr 是什么含义呢？

defStyleAttr 可以理解为是你的 View 的默认风格，例如，你希望你 app 中的所有 MaterialButton 最小高度都为 72dp，那么你可以在你 app 中的 style.xml 中声明:

```xml

<style name="Theme.Demo">
    ...
    <item name="materialButtonStyle">@style/BigButton</item>
</style>

<style name="BigButton" parent="Widget.MaterialComponents.Button">
    <item name="android:minHeight">72dp</item>
</style>
```

这里 MaterialButton 构造时使用的 defStyleAttr 即`R.attr.materialButtonStyle`, 我们在`Theme.Demo`的 style 配置中定义了我们自己的`materialButtonStyle`, 因此 App 内所有用的 MaterialButton 最小高度就是 72dp 了

实际上在上一个 Part 中我们提到`Theme.obtainStyledAttributes(android.util.AttributeSet, int[], int, int)`时会发现其第三个参数名也是 defStyleAttr, 这里的`defStyleAttr`和我们在 View 的构造函数里面表达的是同一个意思. 这里举个例子方便大家理解, 以 Android 的 TextView 举例:

首先我们需要声明一下我们 TextView 支持的属性:

```xml
<resources>
  <declare-styleable name="Theme">

    <!-- ...snip... -->

    <!-- Default TextView style. -->
    <attr name="textViewStyle" format="reference" />

    <!-- ...etc... -->

  </declare-styleable>
</resource>

```

这里的 textViewStyle 被我们声明为 reference, 即 textViewStyle 可以被定义为某一种 style

接着我们声明一下我们的 style

```xml

<resources>
  <style name="Theme">

    <!-- ...snip... -->

    <item name="textViewStyle">@style/Widget.TextView</item>

    <!-- ...etc... -->

  </style>
</resource>
```

在 Manifest 中我们声明使用此 style

```xml
<activity
  android:name=".MyActivity"
  android:theme="@style/Theme"
  />
```

接着在 TextView 构造时我们调用:

```java
TypedArray ta = theme.obtainStyledAttributes(attrs, R.styleable.TextView, R.attr.textViewStyle, 0);
```

此时会产生如下的结果:

1. 任何在 xml 中显式定义的属性会优先返回
2. 如果 xml 中没有定义的属性, 那么会从`R.styleable.TextView`指向的 style, 在这里即`@style/Widget.TextView`, 取出

### defStyleRes

View 的构造函数中还有最后一种: `View(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)`, 其中的 defStyleRes 又是什么意思呢？

简单来说, 当 defStyleAttr 为 0 或者没有在当前主题中设置 defStyleAttr 时, 就会使用 defStyleRes 对应的 style 来构造这个 View

## 小结

列举一下在不同位置设置的属性使用的优先级，简单来说优先级从高到低如下排列:

    1. Any value defined in the AttributeSet.
    2. The style resource defined in the AttributeSet (i.e. style=@style/blah).
    3. The default style attribute specified by defStyleAttr.
    4. The default style resource specified by defStyleResource (if there was no defStyleAttr).
    5. Values in the theme.

## P.S 为什么不使用集联方式实现自定义 View

这里有一点需要注意, 往往我们会采用如下方式实现自定义 View:

```java
class MyView extends SomeView {
    public MyView(Context context) {
        this(context, null);
    }

    public MyView(Context context, AttributeSet attrs) {
        this(context,attrs, 0);
    }

    public MyView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        // do something cool
    }
}
```

这种写法被称为`telescopic constructor`

但这样写有可能会丢掉继承自 SomeView 的默认风格配置, 因为很有可能 SomeView 就是用集联方式实现构造的, 例如:

```java

public SomeView(Context context) {
    this(context, null);
}

public SomeView(Context context, @Nullable AttributeSet attrs) {
    this(context, attrs, com.android.internal.R.attr.textViewStyle);
}

public SomeView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    this(context, attrs, defStyleAttr, 0);
}
```

如果你在`MyView(Context)`中没有调用`super(context)`, 那么你就会丢掉`com.android.internal.R.attr.textViewStyle`的所有默认属性值, 会给你造成一些麻烦, 因此建议参考[TL;DR](#tldr)中的方式实现自定义 View 的构造

## References

[Stackoverflow](https://stackoverflow.com/a/34301725/4380801)
[A deep dive into Android View constructors](https://blog.danlew.net/2016/07/19/a-deep-dive-into-android-view-constructors/)
[Resolving View Attributes on Android](https://ataulm.github.io/2019/10/28/resolving-view-attributes.html)
