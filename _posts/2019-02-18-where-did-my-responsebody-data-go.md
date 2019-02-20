---
layout:     post
title:      Where Did My ResponseBody Data Go?
date:       2019-02-18 14:00:00
summary:    The case of lost bytes in a Retrofit Converter.
categories: Android
---
I am writing an [answer for a Stack Overflow question](https://stackoverflow.com/a/53640537/1696171), and I have with a bug that is not immediately obvious to me.

I want to skip a prefix in an okhttp3.ResponseBody, so that I can forward this body on to be handled without the prefixed bytes. I have code like the following.
~~~
void skipUnusedBodyPrefix(ResponseBody value) {
  BufferedSource source = value.source();
  source.skip(source.indexOf(JSON_PREFIX) + 1);
}
~~~
To my surprise, I receive a `java.io.EOFException: End of input` when I read the body after skipping the prefix. My body is read somewhere too early.

I start with some knowledge about Okio types. True to its name, a BufferedSource buffers. A BufferedSource reads a segment's size (8KB, but that is an implementation detail) into its internal buffer. With this knowledge, I check and confirm that my small (<8KB) response body is being buffered into my ResponseBody's BufferedSource.

Then, I realize that my ResponseBody is actually creating a new BufferedSource every time `.source()` is called on it. Thus, when I pass off my ResponseBody and later call .source() on it to use it, the newly created ResponseBody has none of the body's already buffered data. The body's buffered data is lost back in `skipUnusedBodyPrefix`. This surprises me because .source() [usually](https://github.com/square/okhttp/blob/cdacead7fa826e00825443ebb4dc71f48dd35f4c/okhttp/src/main/java/okhttp3/internal/http/RealResponseBody.java#L48) returns a single BufferedSource cached on the ResponseBody. However, ResponseBody's API does not require such behavior. In this case, [Retrofit's ResponseBody returns a new BufferedSource instance](https://github.com/square/retrofit/blob/8839c4fc4e859e458296400123a6bb97a0985283/retrofit/src/main/java/retrofit2/OkHttpCall.java#L295) for every .source() call.

To fix this, I create a new ResponseBody that reuses my BufferedSource and passes this ResponseBody on to be used.
~~~
ResponseBody withoutUnusedBodyPrefix(ResponseBody value) {
  BufferedSource source = value.source();
  source.skip(source.indexOf(JSON_PREFIX) + 1);
  return ResponseBody.create(
      value.contentType(), value.contentLength(), source);
}
~~~
Fixed. We skip the prefixed bytes, pass the new ResponseBody on to be read, and our solution works now.

Knowing a little Okio helps, even when we application-layer developers normally only touch high-level abstractions like Retrofit converters.
