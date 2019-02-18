---
layout:     post
title:      Moshi in 2 parts
date:       2018-09-11 23:00:00
summary:    Moshi's types API is built on top of its streaming API.
categories: Android
---
[Moshi](https://github.com/square/moshi/) has APIs for reading JSON off a BufferedSource and writing JSON to a BufferedSink and has a second set of APIs for mapping JSON to Java types.

[JsonReader](https://square.github.io/moshi/1.x/moshi/com/squareup/moshi/JsonReader.html) and [JsonWriter](https://square.github.io/moshi/1.x/moshi/com/squareup/moshi/JsonWriter.html) are the former, the streaming APIs. [Moshi](https://square.github.io/moshi/1.x/moshi/com/squareup/moshi/Moshi.html), [JsonAdapter](https://square.github.io/moshi/1.x/moshi/com/squareup/moshi/JsonAdapter.html), and the [Types factory methods](https://square.github.io/moshi/1.x/moshi/com/squareup/moshi/Types.html) are the latter, the object-mapping APIs.

Moshi's Java object-mapping APIs are built on top of the streaming APIs. In fact, one could almost build an object-mapping API for another JVM (or perhaps even [non-JVM soon](https://medium.com/square-corner-blog/okio-2-6f6c35149525/)) language on top of Moshi's public streaming APIs. There is an exception.

Moshi's built-in adapter for maps uses an unexposed part of the streaming API. The map adapter tells readers and writers to treat JSON values as JSON keys. This allows any JsonAdapter to be used for JSON keys, so the map adapter can handle maps with non-string key types. Note that the JsonAdapter used for map keys must serialize the key type to a string or number, so the value can be converted to a string for the JSON object's key.

Other than this case, Moshi is two libraries.
