---
layout:     post
title:      Polymorphic Deserialization with Moshi
date:       2018-09-11 23:30:00
summary:    Moshi can map raw Java types to custom Java models.
categories: Android
---
<b>NOTE: The updated preferred method of polymorphic deserialization is in a [new post](/peeking-streams-with-moshi).</b>

[Moshi](https://github.com/square/moshi/) has the machinery to map raw Java types (maps, lists, strings, numbers, booleans, and nulls) to custom Java models. Since these raw Java types represent JSON types, a major benefit for JSON parsing is support for "polymorphic" object-mapping. When a JSON model represents different Java models depending on the JSON's contents, the JSON can be read in as raw Java types. The raw Java values can be inspected and then can be mapped to the appropriate custom Java models.

Here is a common example.
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
static Vehicle readVehicle(JsonReader reader, JsonAdapter<Vehicle.Train> trainAdapter,
    JsonAdapter<Vehicle.Sailboat> sailboatAdapter, JsonAdapter<Vehicle.Rocket> rocketAdapter) {
  Object rawValue = reader.readJsonValue();
  String type = (String) ((Map<String, Object>) rawValue).get("type");
  switch (type) {
    case "🚂":
      return trainAdapter.fromJsonValue(rawValue);
    case "⛵️":
      return sailboatAdapter.fromJsonValue(rawValue);
    case "🚀":
      return rocketAdapter.fromJsonValue(rawValue);
    default:
      throw new JsonDataException("Unexpected type: " + type);
  }
}
~~~

In almost all cases, this works. There is a very rare case where this will break. Note that `JsonReader.nextString()` reads unquoted numbers as strings to preserve precision for large numbers. Also note that `JsonReader.readJsonValue` reads unquoted numbers as doubles.
Consider this JSON.
~~~
{
  "type": "🚀",
  "mps": 9223372036854775808
}
~~~
`JsonReader.readJsonValue` will read this number as a double, losing precision.
~~~
Moshi moshi = new Moshi.Builder().build();
JsonAdapter<Vehicle.Train> trainAdapter = moshi.adapter(Vehicle.Train.class);
JsonAdapter<Vehicle.Sailboat> sailboatAdapter = moshi.adapter(Vehicle.Sailboat.class);
JsonAdapter<Vehicle.Rocket> rocketAdapter = moshi.adapter(Vehicle.Rocket.class);
String json = ""
    + "{\n"
    + "  \"type\": \"🚀\",\n"
    + "  \"mps\": 9223372036854775808\n"
    + "}";
JsonReader reader = JsonReader.of(new Buffer().writeUtf8(json));
Vehicle vehicle = readVehicle(reader, trainAdapter, sailboatAdapter, rocketAdapter);

Vehicle.Rocket rocket = (Vehicle.Rocket) vehicle;
System.out.println(rocket.mps); // 9.223372036854776E18 Lost precision!
~~~

Generally, putting numbers in JSON that cannot be represented as doubles in Java is a bad idea. Even so, we can fix this.
Instead of using `JsonReader.readJsonValue` to decode into the raw Java types, let's read the JSON ourselves. When we reach a number, we can use a `BigDecimal` instead of a `double`. So that we do not have to rewrite all of the bindings to Java's raw types ourselves, we can use Moshi's built-in adapter for Object.
~~~
static final JsonAdapter.Factory OBJECT_JSON_ADAPTER_FACTORY = new JsonAdapter.Factory() {
  @Nullable @Override
  public JsonAdapter<?> create(Type type, Set<? extends Annotation> annotations, Moshi moshi) {
    if (type != Object.class) {
      return null;
    }
    JsonAdapter<Object> delegate = moshi.nextAdapter(this, Object.class, emptySet());
    return new JsonAdapter<Object>() {
      @Override public @Nullable Object fromJson(JsonReader reader) throws IOException {
        if (reader.peek() != JsonReader.Token.NUMBER) {
          return delegate.fromJson(reader);
        } else {
          return new BigDecimal(reader.nextString());
        }
      }
    };
  }
};

static Vehicle readVehicle(JsonReader reader, JsonAdapter<Object> objectAdapter, JsonAdapter<Vehicle.Train> trainAdapter,
    JsonAdapter<Vehicle.Sailboat> sailboatAdapter, JsonAdapter<Vehicle.Rocket> rocketAdapter)
    throws IOException {
  Object rawValue = objectAdapter.fromJson(reader);
  String type = (String) ((Map<String, Object>) rawValue).get("type");
  switch (type) {
    case "🚂":
      return trainAdapter.fromJsonValue(rawValue);
    case "⛵️":
      return sailboatAdapter.fromJsonValue(rawValue);
    case "🚀":
      return rocketAdapter.fromJsonValue(rawValue);
    default:
      throw new JsonDataException("Unexpected type: " + type);
  }
}

Moshi moshi = new Moshi.Builder().add(OBJECT_JSON_ADAPTER_FACTORY).build();
JsonAdapter<Object> objectAdapter = moshi.adapter(Object.class);
JsonAdapter<Vehicle.Train> trainAdapter = moshi.adapter(Vehicle.Train.class);
JsonAdapter<Vehicle.Sailboat> sailboatAdapter = moshi.adapter(Vehicle.Sailboat.class);
JsonAdapter<Vehicle.Rocket> rocketAdapter = moshi.adapter(Vehicle.Rocket.class);
String json = ""
    + "{\n"
    + "  \"type\": \"🚀\",\n"
    + "  \"mps\": 9223372036854775808\n"
    + "}";
JsonReader reader = JsonReader.of(new Buffer().writeUtf8(json));

Vehicle vehicle = readVehicle(reader, objectAdapter, trainAdapter, sailboatAdapter,
    rocketAdapter);
Vehicle.Rocket rocket = (Vehicle.Rocket) vehicle;
System.out.println(rocket.mps); // 9223372036854775808 Correct.
~~~
Now, we can use this string as a number in big calculations.

To generalize the common case of polymorphic JSON deserialization and reduce error-prone code, a [RuntimeJsonAdapterFactory](https://github.com/square/moshi/blob/fe22970973ebe8c2273786f095b43022a87ef8e8/adapters/src/main/java/com/squareup/moshi/adapters/RuntimeJsonAdapterFactory.java) is provided in Moshi's adapters module. If you need to handle big numbers, consider adding a custom Object JsonAdapter like in the above example to your Moshi instance.
