---
layout:     post
title:      Reading file types is easy with Okio
date:       2017-02-08 08:00:00
summary:    An example of getting the MediaType from an image file.
categories: Android
---
I recently had photos I needed to upload to my server.
To enter the correct MIME type into the Content-Type header of my request, I need to know the file type.

I had a general idea of how to inspect the first few bytes for the file's [magic number](https://en.wikipedia.org/wiki/File_format#Magic_number), but I decided to do a few searches online to see how others find file types in Java.
Most of the approaches I saw relied on obscure, very large libraries to handle the implementation, and a few stopped at the predictable hack of parsing the file name.

Fortunately, [Okio](https://github.com/square/okio) makes finding the file type extremely easy. (And, every modern Java developer is already using Okio.)

Here is an example that specifically handles pngs and jpegs and creates OkHttp's MediaType for my request.
To fill this sample in more completely, check out more magic numbers in the [Wikipedia list](https://en.wikipedia.org/wiki/List_of_file_signatures).

~~~
private static final ByteString PNG_HEADER = ByteString.decodeHex("89504e470d0a1a0a");
private static final ByteString JPEG_HEADER = ByteString.decodeHex("ffd8ff");

static MediaType mediaType(File media) throws IOException {
  BufferedSource mediaSource = Okio.buffer(Okio.source(media));
  try {
    if (mediaSource.rangeEquals(0, PNG_HEADER)) {
      return MediaType.parse("image/png");
    } else if (mediaSource.rangeEquals(0, JPEG_HEADER)) {
      return MediaType.parse("image/jpeg");
    } else {
      throw new IOException("Unsupported file type.");
    }
  } finally {
    try {
      mediaSource.close();
    } catch (IOException ignored) {
    }
  }
}
~~~
