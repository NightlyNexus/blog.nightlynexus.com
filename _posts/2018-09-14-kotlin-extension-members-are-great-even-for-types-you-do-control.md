---
layout:     post
title:      Kotlin Extension Members Are Great Even for Types You Do Control
date:       2018-09-14 23:30:00
summary:    Extensions can add more bounds on generics.
categories: Android
---

Kotlin extension members are great. When using a type that you do not control, you can paper over the awkwardness of calling a static utility method and promote the member to appear as an instance member of the type, with full discoverability in your IDE's autocomplete.

Kotlin extension functions have a use even for types that you **do** control!

Consider this case:
~~~
abstract class TypeAdapter<T> { // Implicit <T: Any?>
  abstract fun adapt(string: String): T

  fun nonNull(): TypeAdapter<T> {
    val delegate = this
    return object: TypeAdapter<T>() {
      override fun adapt(string: String): T {
        return delegate.adapt(string)!!
      }
    }
  }
}
~~~
This is unfortunate for the API.
~~~
object IdentityTypeAdapter: TypeAdapter<String>() {
  override fun adapt(string: String) = string
}
~~~
With the current API of `TypeAdapter`, we can call `IdentityTypeAdapter.nonNull()`, even though `IdentityTypeAdapter`'s `adapt` function never returns null, anyway! `IdentityTypeAdapter.nonNull()` is a useless factory call.

An extension function can improve the API by limiting the scope of the generic type argument for the `nonNull` function.
~~~
abstract class TypeAdapter<T> { // Implicit <T: Any?>
  abstract fun adapt(string: String): T
}

fun <T: Any> TypeAdapter<T?>.nonNull(): TypeAdapter<T> {
  val delegate = this
  return object: TypeAdapter<T>() {
    override fun adapt(string: String): T {
      return delegate.adapt(string)!!
    }
  }
}
~~~
This new API uses an extension function on its own type to constrain the generic type argument's bounds further for just this `nonNull` function.
Now, `IdentityTypeAdapter.nonNull()` will not compile. `nonNull()` is only a valid function for `TypeAdapter`s with nullable type arguments.


Of course, we could add similar extension functions in our API for other bounds like
~~~
fun TypeAdapter<Number>.adaptToSuccessOrFailure(string: String): Boolean {
  return adapt(string) == 0
}
~~~
just as we could in Java with static methods like
~~~
static boolean adaptToSuccessOrFailure(TypeAdaper<Number> adapter, String string) {
  return adapter.adapt(string) == 0;
}
~~~
The Java API is hidden and awkward.
<br/>With Kotlin, add the extension function to your API!
