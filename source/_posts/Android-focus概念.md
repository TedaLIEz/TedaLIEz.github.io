---
title: Android focus概念
date: 2020-02-14 17:52:43
tags: android
---

# Android focus 相关概念

## focus

焦点实际上是窗口系统中一个重要概念，如果一个窗体获得了焦点，其含义是该窗体获得了可交互的机会。在 android 中也是如此，例如一个 EditText，只有当其获得了焦点时，我们才能通过键盘对其做输入操作

## focusable && focusableInTouchMode

这两个属性是比较容易造成理解混淆。

首先我们看下`focusableInTouchMode`的官方定义

    Boolean that controls whether a view can take focus while in touch mode. If this is true for a view, that view can gain focus when clicked on, and can keep focus if another view is clicked on that doesn't have this attribute set to true.

翻译一下，如果设置为 true，那么当其被触摸点击时，就有能力获得焦点；并且其他为 false 的 View 的点击都不能夺取这个焦点。

举例来说，如果有一个 EditText 其`focusableInTouchMode`为 true, 那么我们在屏幕上对其点击就会拉起虚拟键盘。

另外，有些业务场景是点击 EditText 后不拉起键盘，而是弹起 Dialog 来输入的场景。那么这个时候最简单的方案就是设置`focusableInTouchMode`为 false，然后给这个 EditText 的点击事件实现你的弹窗逻辑

但这里就有个冲突，如果`focusableInTouchMode`为 true，那么此时我对 Widget 的点击到底算是点击(click)事件还是算作 focus 事件呢？答案是算作 focus 事件，所以有时候你会遇到给 Button 设置`focusableInTouchMode`为 true 后，每次第一次点击都不触发 onClick 的问题， 因为其 focus 的逻辑会优先 click 逻辑触发

事实上，任何窗体系统的交互其实是要分为两步走的:

1. 找到特定的 Widget，让其获得焦点
2. 开始交互(输入, 点击等)

因此获取焦点的逻辑应该都是优先触发的

这里有个疑问是为什么额外需要一个 touchMode，我的理解是跟 android 机器面临的历史问题，历史上 android 机器曾经是有软硬件两种操作方式的。有些古老手机上会保留类似滚球的操作设备来在不同窗体上 focus。另外，类似智能电视这样的设备上也会有硬件(遥控器), 触摸屏两种交互的方式.

而 focusable 则是表示一个 Widget 是否能被 focus，如果这个属性设置成 false，那么这个 Widget 是不能获得焦点的，用户也无法与这个 Widget 进行交互(例如输入)

同时根据文档描述

    Setting this to false will also ensure that this view is not focusable in touch mode.

这个属性为 false 时，会将`focusableInTouchMode`也变成 false

在代码中，我们发现`View#setFocusable`有一个 int 作为参数，除去两个常见的`FOCUSABLE`和`NOT_FOCUSABLE`对应 xml 中的 true 和 false 外，我们看到其还有一个选项, `FOCUSABLE_AUTO`. 根据注释，这个是 Widget 的默认 focusable 值，android 系统会自动决定 Widget 的焦点

## descendantFocusability 属性

这个属性常常用在 ViewGroup，用来定义这个 ViewGroup 和其子 View 之间获取焦点时的关系

```
+------------------------------------------------------------------------------------------+
|      Constant            Value            Description                                    |
+------------------------------------------------------------------------------------------+
| afterDescendants           1          The ViewGroup will get focus only if               |
|                                       none of its descendants want it.                   |
+------------------------------------------------------------------------------------------+
| beforeDescendants          0          The ViewGroup will get focus before                |
|                                       any of its descendants.                            |
+------------------------------------------------------------------------------------------+
| blocksDescendants          2          The ViewGroup will block its descendants from      |
|                                       receiving focus.                                   |
+------------------------------------------------------------------------------------------+
```

也就是说，根据不同的定义，ViewGroup 有能力在子 View 之前，之后，或者阻止子 View 获取焦点

那么这个属性的使用场景是什么呢? 例如你要在一个 ListView 里面与 item 的某个元素做交互, 那么你是希望子 View 先于 ListView 交互的(这样才能避免 ListView 的焦点逻辑优先触发), 此时设置 afterDescendants 就是有用的

```java
public void onItemSelected(AdapterView<?> parent, View view,
        int position, long id) {
    ListView listView = getListView();
    Log.d(TAG, "onItemSelected gave us " + view.toString());
    Button b = (Button) view.findViewById(R.id.button);
    EditText et = (EditText) view.findViewById(R.id.editor);
    if (b != null || et != null) {
        // Use afterDescendants to keep ListView from getting focus
        listView.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        if(et!=null) et.requestFocus();
        else if(b!=null) b.requestFocus();
    } else {
        if (!listView.isFocused()) {
            // Use beforeDescendants so that previous selections don't re-take focus
            listView.setDescendantFocusability(ViewGroup.FOCUS_BEFORE_DESCENDANTS);
            listView.requestFocus();
        }
    }

}
```
