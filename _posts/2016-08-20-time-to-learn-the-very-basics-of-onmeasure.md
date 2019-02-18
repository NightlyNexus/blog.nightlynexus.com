---
layout:     post
title:      A simple implementation of onMeasure
date:       2016-08-20 09:08:19
summary:    Implementing a SquareFrameLayout.
categories: Android
---
Custom Android Views like to override [onMeasure](https://developer.android.com/reference/android/view/View.html#onMeasure(int,%20int)) for a variety of reasons.
Let's look at an easy and common use case.

We want to create a FrameLayout that is always square.
The first attempt to do so usually looks like the following:

{% highlight java %}
public final class SquareFrameLayout extends FrameLayout {
  @Override protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int size = Math.min(getMeasuredWidth(), getMeasuredHeight());
    setMeasuredDimension(size, size);
  }
}
{% endhighlight %}

This might appear to work fine in some cases, but let’s look at the following:

{% highlight xml %}
<SquareFrameLayout
  android:layout_width="320dp"
  android:layout_height="200dp"
  android:background="#FFF">
  <ImageView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:src="@mipmap/ic_launcher"/>
</SquareFrameLayout>
{% endhighlight %}

Here is what that looks like:

![](/images/squareframelayout-incorrect.jpg)

`setMeasuredDimension` just sets the result of `onMeasure`; it does not affect how children are measured. The child ImageView is told that its width should each be exactly 320 dips and measures itself accordingly. After that, our SquareFrameLayout decides to change its measurement result.\\
Also, our SquareFrameLayout really should honor its width and height and only change its children's widths and heights.

The real solution for this problem involves calculating the appropriate size and making the right measure specs for `super.onMeasure`:

{% highlight java %}
@Override protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  int widthMode = MeasureSpec.getMode(widthMeasureSpec);
  int heightMode = MeasureSpec.getMode(heightMeasureSpec);
  int widthSize = MeasureSpec.getSize(widthMeasureSpec);
  int heightSize = MeasureSpec.getSize(heightMeasureSpec);
  float widthToHeightRatio = 1f; // Is a square.
  int width;
  int height;
  if (widthMode == UNSPECIFIED && heightMode == UNSPECIFIED
      || widthMode != UNSPECIFIED && heightMode != UNSPECIFIED) {
    if (widthSize / widthToHeightRatio > heightSize) {
      // Limited by height.
      width = (int) (heightSize * widthToHeightRatio);
      height = heightSize;
    } else {
      // Limited by width.
      width = widthSize;
      height = (int) (widthSize / widthToHeightRatio);
    }
  } else if (widthMode == UNSPECIFIED) {
    // Limited by height.
    width = (int) (heightSize * widthToHeightRatio);
    height = heightSize;
  } else {
    // Limited by width.
    width = widthSize;
    height = (int) (widthSize / widthToHeightRatio);
  }
  super.onMeasure(MeasureSpec.makeMeasureSpec(width, EXACTLY),
  MeasureSpec.makeMeasureSpec(height, EXACTLY));
  if (widthMode == EXACTLY && heightMode == EXACTLY) {
    setMeasuredDimension(widthSize, heightSize);
  } else if (widthMode == EXACTLY) {
    setMeasuredDimension(widthSize, getMeasuredHeight());
  } else if (heightMode == EXACTLY) {
    setMeasuredDimension(getMeasuredWidth(), heightSize);
  }
}
{% endhighlight %}

Run the above layout, and we will see the child Views and our SquareLinearLayout (shown with the white background) properly sized:

![](/images/squareframelayout-correct.jpg)

If you really want to change the size of the SquareFrameLayout and not just the size of its children, you can alter the resulting setMeasuredDimension calls at the end of `onMeasure`.

For the basics of MeasureSpec and how Android measures, lays out, and draws Views, [the introduction documentation](https://developer.android.com/guide/topics/ui/how-android-draws.html) is a great read. It is only a few paragraphs long; I wish I had read it sooner!
