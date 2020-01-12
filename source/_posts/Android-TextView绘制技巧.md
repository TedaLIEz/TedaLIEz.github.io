---
title: Android TextView绘制技巧
date: 2020-01-12 09:22:37
tags: android, textview
---

## 文本绘制概念

- Top - The maximum distance above the baseline for the tallest glyph in the font at a given text size.
- Ascent - The recommended distance above the baseline for singled spaced text.
- Descent - The recommended distance below the baseline for singled spaced text.
- Bottom - The maximum distance below the baseline for the lowest glyph in the font at a given text size.
- Leading - The recommended additional space to add between lines of text.

{% asset_img android_textview.png %}

这套概念在`android.text.Layout`中也成立

需要注意的是 android.text.Layout 也是从 0 计算的, 对于其 API getLineBottom, getLineBaseline, getLineTop.... 都是从 0 计算的

同时, android.text.TextView#getLayout()方法必定在 measure 过程结束后才能非空

## TextView#Layout 奇招

TextView 的 Layout 属性管理了文本的布局展示，同时也提供了两个能力，

获取省略号在当前行的开始位置；
获取当前行内被省略文字的长度。

    /** * Return the offset of the first character to be ellipsized away, * relative to the start of the line. (So 0 if the beginning of the * line is ellipsized, not getLineStart().) */
    public abstract int getEllipsisStart(int line);

    /** * Returns the number of characters to be ellipsized away, or 0 if * no ellipsis is to take place. */
    public abstract int getEllipsisCount(int line);

有了以上能力，针对上文最后提到的问题马上便会有思路：通过精确计算被省略的文字位置，截取字符串重新插入占位标识符，然后实现在省略号处添加图片

```java
String text = guessLikeBean.getTitle()+"(精)";
mTvTitle.setText(text);
int ellipsisCount = mTvTitle.getLayout().getEllipsisCount(mTvTitle.getLineCount() - 1);
if (ellipsisCount > 0) {
    text = text.substring(0, text.length() - ellipsisCount - 1) + "…(精)";
}
SpannableString imageString = new SpannableString(text);
...
```

同时，为了保证 TextView#getLayout 一定有值，我们可以使用 View#post 函数来调用我们的函数，View#post 会确保 View 在 attach 之后再执行，而 attach 一定保证了 layout 过程先执行

实际上 TextView#Layout 中可以找到很多跟文本相关的属性，在排版的过程中或多或少你会用到，因此遇到文本排版问题时，不妨先试试这个, 例如:

Layout 的一个子类`BoringLayout`,有一个静态方法`isBoring()`,可以用来判断一段文字是否能在一行放下，这个方法就有广泛的应用场景

## 如何绘制居中的图标

参考[https://gist.github.com/TedaLIEz/97f03a2cb842621f022bd27ba1dfd020](https://gist.github.com/TedaLIEz/97f03a2cb842621f022bd27ba1dfd020)

## 如何判断文字能否塞得下一行

```Kotlin
/**
 * titleTxt: 文本
 * layout: TextView#Layout
 * title: TextView
 * maxWidth: 期望的view最大宽度
 * spacing: 期望能够剩余出来的宽度, 即文本的最大宽度为maxWidth - spacing
**/
private fun badgeHelper(titleTxt: String, layout: Layout, title: TextView, maxWidth: Int, spacing: Int) : String {
    var txt = titleTxt + "精精精"
    val lineCount = layout.lineCount
    if (lineCount == 1) {
        Log.d(TAG, "we have only one line text $titleTxt, return")
        return txt
    }
    val dWidth = spacing
    val lastLineStart = layout.getLineStart(lineCount - 1)
    val lastLineStr = txt.replace('.', 'x').substring(lastLineStart)
    val boring = BoringLayout.isBoring(lastLineStr, title.paint)
    if (boring != null) {
        val sW = ScreenUtil.getScreenWidth(title.context)
        val px = maxWidth
        val width = boring.width
        if (sW < px + width) {
            // text ellipsize
            Log.w(TAG, "BoringLayout says it will ellipsize, skip some text now")
            val diff = px + width - sW
            val diffSize = ((diff + dWidth) / title.textSize).roundToInt()
            txt = titleTxt.substring(0, titleTxt.length - diffSize) + title.context.getString(R.string.ellipsize_string) + "(精)"
            Log.d(TAG, "we change txt from $titleTxt to $txt")
            return txt
        }
    }
    val size = (dWidth / title.textSize).roundToInt()
    val seq = TextUtils.ellipsize(lastLineStr, title.paint, (title.width - title.paddingRight - title.paddingLeft).toFloat(), TextUtils.TruncateAt.END)
    val lastIndexOfDot = seq.lastIndexOf(title.context.getString(R.string.ellipsize_string))
    if (lastIndexOfDot != -1 && lastLineStart + lastIndexOfDot - size - 3 > 0) {
        Log.w(TAG, "TextUtils says it will ellipsize, skip some text now")
        txt = titleTxt.substring(0, lastLineStart + lastIndexOfDot - size - 3) + title.context.getString(R.string.ellipsize_string) + "(精)"
        Log.d(TAG, "we change txt from $titleTxt to $txt")
        return txt
    }

    val ellipsisCount = layout.getEllipsisCount(lineCount - 1)
    if (ellipsisCount > 0) {
        Log.w(TAG, "Layout says it will ellipsize, skip some text now")
        txt = txt.substring(0, txt.length - ellipsisCount - size - 3) + title.context.getString(R.string.ellipsize_string) + "(精)"
        Log.d(TAG, "we change txt from $titleTxt to $txt")
        return txt
    }
    return txt

}
```

## References

ref: [https://stackoverflow.com/a/27631737/4380801](https://stackoverflow.com/a/27631737/4380801)

ref: [https://tristanzeng.github.io/2019/05/29/TextView%E5%A4%9A%E8%A1%8C%E6%96%87%E5%AD%97%E8%B6%85%E5%87%BA%E6%97%B6%E5%A6%82%E4%BD%95%E5%9C%A8%E7%9C%81%E7%95%A5%E5%8F%B7%E5%90%8E%E6%B7%BB%E5%8A%A0%E5%9B%BE%E6%A0%87/](https://tristanzeng.github.io/2019/05/29/TextView%E5%A4%9A%E8%A1%8C%E6%96%87%E5%AD%97%E8%B6%85%E5%87%BA%E6%97%B6%E5%A6%82%E4%BD%95%E5%9C%A8%E7%9C%81%E7%95%A5%E5%8F%B7%E5%90%8E%E6%B7%BB%E5%8A%A0%E5%9B%BE%E6%A0%87/)
