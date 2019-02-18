---
layout:     post
title:      Optimized Android Parcelable Reading and Writing
date:       2016-06-24 13:42:10
summary:    Use Parcelable.Creator.createFromParcel(...) and Parcelable.writeToParcel(...) instead of Parcel.readParcelable(...) and Parcel.writeParcelable(...).
categories: Android
---
To save unnecessary reading, writing, and reflection, use `Parcelable.Creator.createFromParcel(...)` and `Parcelable.writeToParcel(...)` instead of `Parcel.readParcelable(...)` and `Parcel.writeParcelable(...)`.

As Android developers who want our apps to survive configuration changes and process death, we use many value classes that implement [Parcelable](https://developer.android.com/reference/android/os/Parcelable.html).

We have written classes like `🚂`:

~~~ java
final class Train implements Parcelable {
  final Manifest manifest;
  final int maxPassengers;

  Train(Manifest manifest, int maxPassengers) {
    this.manifest = manifest;
    this.maxPassengers = maxPassengers;
  }

  Train(Parcel in) {
    manifest = Manifest.CREATOR.createFromParcel(in);
    maxPassengers = in.readInt();
  }

  @Override public void writeToParcel(Parcel dest, int flags) {
    manifest.writeToParcel(dest, flags);
    dest.writeInt(maxPassengers);
  }

  @Override public int describeContents() {
    return 0;
  }

  public static final Creator<Train> CREATOR = new Creator<Train>() {
    @Override public Train createFromParcel(Parcel in) {
      return new Train(in);
    }

    @Override public Train[] newArray(int size) {
      return new Train[size];
    }
  };
}
~~~

In 2016, we have some terrific tools that can generate the boilerplate in such classes.
Android Studio has a built-in live template, and [AutoValue](https://github.com/google/auto/tree/master/value) has a [maintained extension](https://github.com/rharter/auto-value-parcel).

But, there is an optimization that these tools do not take advantage of.

In the above `Train` example, note the use of `Manifest.CREATOR.createFromParcel(in)` and `manifest.writeToParcel(dest, flags)`. We are directly calling the methods we wrote ourselves.
A naïve approach might be to use `in.readParcelable(Manifest.class.getClassLoader())` and `dest.writeParcelable(manifest, flags)`, but this unnecessarily writes a String (the fully qualified class name of the `Manifest.CREATOR`) to the Parcel and then reads the String and might look up the `Parcelable.Creator` instance with reflection (if it is not cached by Android). [See the source here.](https://android.googlesource.com/platform/frameworks/base/+/4e6a73c16ab35c6a8d7f524fa2dad6b8b822d7a9/core/java/android/os/Parcel.java#2442)
Since we already know the instance of the `Parcelable.Creator` we are going to use, we can optimize this code and use the instance directly, rather than use the lookup.

Note that we need to know the runtime type of the Parcelable field to use the correct `Parcelable.Creator` instance, of course, though non-final Parcelable implementations are a code smell.
Also, note that, **if the inner Parcelable field is nullable,** we should first write a flag to indicate a null value:

~~~ java
@Nullable static <T extends Parcelable> T readParcelable(Parcelable.Creator<T> creator, Parcel in) {
  if (in.readInt() == 0) {
    return null;
  }
  return creator.createFromParcel(in);
}

static <T extends Parcelable> void writeParcelable(@Nullable T value, Parcel dest, int flags) {
  if (value == null) {
    dest.writeInt(0);
    return;
  }
  dest.writeInt(1);
  value.writeToParcel(dest, flags);
}
~~~

This is a tiny but easy manual optimization for our Parcelable implementations.
As for tooling, the Android Studio live template is [not currently able to access class members](https://code.google.com/p/android/issues/detail?id=212427), and the AutoValue extension [would require adding too much complexity to apply this optimization only to fields that are final types](https://github.com/rharter/auto-value-parcel/pull/77).

Parcel provides a [handful of native calls](https://android.googlesource.com/platform/frameworks/base/+/4e6a73c16ab35c6a8d7f524fa2dad6b8b822d7a9/core/java/android/os/Parcel.java#254), but the Android SDK provides quite a few of these convenience APIs (mostly around collections and data structures). Make sure the convenience APIs you are using are the most optimized choices for your code, and even [write your own](https://github.com/NightlyNexus/ViewStatePagerAdapter/blob/8c44a9921f245e31c33d2a6d1b60c41824473ffb/viewstatepageradapter/src/main/java/com/nightlynexus/viewstatepageradapter/ViewStatePagerAdapter.java#L130) when you need to do so.

What are more great Parcelable optimizations I should know about?
Ping me on Twitter [@Eric_Cochran](https://twitter.com/Eric_Cochran) or email me at [eric@nightlynexus.com](mailto:eric@nightlynexus.com).
