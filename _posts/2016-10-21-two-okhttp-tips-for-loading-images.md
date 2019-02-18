---
layout:     post
title:      Two OkHttp Tips for Loading Images
date:       2016-10-21 00:28:44
summary:    Use only relevant client customizations and warm the HTTP response cache.
categories: Android
---
Everyone loading images from the network on Android is probably using [OkHttp](https://github.com/square/okhttp).
<br/>Here are two improvements that are very easy to implement.

### 1. Only use relevant Interceptors and customizations.

Most apps use one OkHttpClient to share state such as the disk cache and async threads.
<br/>But, many apps network with multiple hosts, especially when downloading images. This might be inconvenient or even a security concern when using the same OkHttpClient for all these requests.
<br/>In a simple example, we may have an Interceptor like:

~~~
final class RedditAccessTokenInterceptor implements Interceptor {
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    HttpUrl url = request.url().newBuilder().addQueryParameter("access-token", redditAccessToken).build();
    Request transformed = request.newBuilder().url(url).build();
    return chain.proceed(transformed);
  }
}
~~~

The above adds a query parameter to all the requests from the client that it is set on, even when we are getting our pictures from Imgur.
<br/>The solution is simple. Create a single OkHttpClient instance which is not used directly and then call `.newBuilder()` for each use-case customization.

~~~
OkHttpClient unusedBase = new OkHttpClient.Builder()/* General settings. */.build();
OkHttpClient redditClient = unusedBase.newBuilder().addInterceptor(redditAccessTokenInterceptor).build();
OkHttpClient imgurClient = unusedBase/* Other custom settings. */.build();
~~~

Now, thanks to the ease of the builder pattern, our OkHttpClients will share everything except for the per-host customizations.

### 2. Warm the HTTP response cache.

Some apps have many images with known remote locations that display consistently throughout the app.
<br/>Sometimes, these images are too many to put into memory as Bitmaps, but we can still download the images and write the bytes to the HTTP response cache.

~~~
static final Callback CACHING = new Callback() {
  @Override public void onFailure(Call call, IOException e) {}
  @Override public void onResponse(Call call, Response response) throws IOException {
    try (ResponseBody body = response.body()) {
      if (response.cacheResponse() == null) {
        body.source().skip(Long.MAX_VALUE); // Exhaust response body.
      }
    }
  }
};
~~~

We can run our image cache requests frequently to keep our images up to date.
<br/>With the proper cache headers and a large enough maximum cache size, we should almost always hit the response cache and discard the body.
<br/>When we do make the network call, we exhaust and write-out the response body to push and save it to the cache.

We just made all our known images load in much faster and stay updated with very little configuration.

What are some more easy wins that OkHttp lets us take?
<br/>Ping me on Twitter [@Eric_Cochran](https://twitter.com/Eric_Cochran) or email me at [eric@nightlynexus.com](mailto:eric@nightlynexus.com).
