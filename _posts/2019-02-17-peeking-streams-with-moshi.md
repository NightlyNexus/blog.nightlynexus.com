---
layout:     post
title:      Peeking Streams with Moshi
date:       2019-02-17 23:33:00
summary:    Moshi can read over JSON multiple times.
categories: Android
---
Peeking on [Okio](https://github.com/square/okio/)'s BufferedSource allows for reading ahead on streams. [Moshi](https://github.com/square/moshi/) uses this feature to support decoding JSON data multiple times. This is useful for object mapping when the JSON format is unknown. One pass can reveal the structure of the data, and a second pass can use the data. Two common examples are decoding polymorphic JSON and guarding against unexpected JSON formats.

<h2>Polymorphic JSON.</h2>
~~~
abstract class Vehicle {
  final String type;

  static final class Train extends Vehicle {
    final String mph;
  }

  static final class Sailboat extends Vehicle {
    final String knots;
  }

  static final class Rocket extends Vehicle {
    final String mps;
  }
}
~~~
`Train` in JSON is
~~~
{
  "type": "🚂",
  "mph": 60
}
~~~
`Sailboat` in JSON is
~~~
{
  "type": "⛵️",
  "knots": 7
}
~~~
`Rocket` in JSON is
~~~
{
  "type": "🚀",
  "mps": 7900
}
~~~
To read this JSON, we can implement our polymorphic deserialization like this.
~~~
final class VehicleJsonAdapter extends JsonAdapter<Vehicle> {
  static final JsonReader.Options TYPE_OPTIONS = JsonReader.Options.of("type");
  static final JsonReader.Options VEHICLE_OPTIONS = JsonReader.Options.of("🚂", "⛵️", "🚀");
  final JsonAdapter<Vehicle.Train> trainAdapter;
  final JsonAdapter<Vehicle.Sailboat> sailboatAdapter;
  final JsonAdapter<Vehicle.Rocket> rocketAdapter;

  @Override public Vehicle fromJson(JsonReader reader) throws IOException {
    int vehicleOptionsIndex = -1;
    try (JsonReader peeked = reader.peekJson()) {
      peeked.beginObject();
      while (peeked.hasNext()) {
        if (peeked.selectName(TYPE_OPTIONS) == 0) {
          int index = peeked.selectString(VEHICLE_OPTIONS);
          if (index == -1) {
            throw new JsonDataException("Unexpected vehicle type: " + peeked.nextString());
          }
          vehicleOptionsIndex = index;
          break;
        }
        peeked.skipName();
        peeked.skipValue();
      }
    }
    switch (vehicleOptionsIndex) {
      case -1:
        throw new JsonDataException("No type label in the JSON.");
      case 0:
        return trainAdapter.fromJson(reader);
      case 1:
        return sailboatAdapter.fromJson(reader);
      case 2:
        return rocketAdapter.fromJson(reader);
      default:
        throw new AssertionError();
    }
  }
}
~~~
We peek on the reader, find the key for decoding by reading ahead, and then return to do the decoding with the proper object mapping. Note that, for best performance, this "type" key should be the first field in the object. Otherwise, we would reprocess more of the JSON stream than we need to.

Example usage.
~~~
JsonAdapter.Factory vehicleJsonAdapterFactory = new JsonAdapter.Factory() {
  @Override public @Nullable JsonAdapter<?> create(Type type,
      Set<? extends Annotation> annotations, Moshi moshi) {
    if (type != Vehicle.class || !annotations.isEmpty()) {
      return null;
    }
    JsonAdapter<Vehicle.Train> trainAdapter = moshi.adapter(Vehicle.Train.class);
    JsonAdapter<Vehicle.Sailboat> sailboatAdapter = moshi.adapter(Vehicle.Sailboat.class);
    JsonAdapter<Vehicle.Rocket> rocketAdapter = moshi.adapter(Vehicle.Rocket.class);
    return new VehicleJsonAdapter(trainAdapter, sailboatAdapter, rocketAdapter);
  }
};
Moshi moshi = new Moshi.Builder().add(vehicleJsonAdapterFactory).build();
JsonAdapter<Vehicle> vehicleAdapter = moshi.adapter(Vehicle.class);
String encoded = ""
    + "{\n"
    + "  \"type\": \"🚀\",\n"
    + "  \"mps\": 9223372036854775808\n"
    + "}";
Vehicle vehicle = vehicleAdapter.fromJson(encoded);
System.out.println(vehicle);
~~~

To generalize the common case of polymorphic JSON deserialization and reduce error-prone code, a [PolymorphicJsonAdapterFactory](https://github.com/square/moshi/blob/be6f3eb2affcbca1d41a1d396870e052cbbb3bd5/adapters/src/main/java/com/squareup/moshi/adapters/PolymorphicJsonAdapterFactory.java) is provided in Moshi's adapters module.

<h2>Guarding against unexpected JSON formats.</h2>
~~~
final class Camera {
  final @Nullable String manufacturer;

  static final Object JSON_ADAPTER = new Object() {
    @FromJson Camera fromJson(JsonReader reader, JsonAdapter<Camera> delegate) throws IOException {
      Camera camera;
      try (JsonReader peeked = reader.peekJson()) {
        camera = delegate.fromJson(peeked);
      } catch (JsonDataException e) {
        camera = new Camera(null);
      }
      reader.skipValue();
      return camera;
    }
  };
}
~~~
We use a peeked reader to leave the reader in a known state even if there's an exception. Then, we attempt to decode to the Camera type with the peeked reader. Finally, we skip the value back on the reader, no matter the state of the peeked reader.

Example usage.
~~~
Moshi moshi = new Moshi.Builder().add(Camera.JSON_ADAPTER).build();
JsonAdapter<List<Camera>> cameraListAdapter = moshi.adapter(
    Types.newParameterizedType(List.class, Camera.class));
String encoded = ""
    + "[\n"
    + "  {\n"
    + "    \"manufacturer\": \"Canon\"\n"
    + "  },\n"
    + "  \"📷\"\n"
    + "]";
List<Camera> cameras = cameraListAdapter.fromJson(encoded);
System.out.println(cameras);
~~~
For a generalization of this case, see [DefaultOnDataMismatchAdapter](https://github.com/square/moshi/blob/a83a9b79e43fa3fcc6988e0a48bb32341ec1b68c/examples/src/main/java/com/squareup/moshi/recipes/DefaultOnDataMismatchAdapter.java).
