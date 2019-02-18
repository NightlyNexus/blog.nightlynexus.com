---
layout:     post
title:      Guidelines I Use When Naming Code Things
date:       2018-09-13 23:55:00
summary:    Do not use abbreviations, use qualifiers, and remove duplication.
categories: Android
---
Do not use abbreviations.
Abbreviations harm searchability.

Put qualifiers first in names. In English, modifiers precede the things they describe. For an implementation of a `Source`, instead of `SourceBuffered`, prefer `BufferedSource`. For an implementation of a `BufferedSource`, instead of `BufferedSourceImpl` (also an abbreviation!), prefer `RealBufferedSource`.

Do not leak implementation details into API names.
Instead of exposing a `BaseDemoMode`, `AbstractDemoMode`, or `IDemoMode` (also an abbreviation!), expose a `DemoMode` and make the internal implementation have the qualified name, `RealDemoMode`.
For real implementations of types that are intended to have fakes and have no specific qualifier, use the prefix `Real`, as in `RealDemoMode`.
"Real" is the opposite of "Fake," which is a good prefix for fake implementations in tests.

Remove redundancy.

Use class name qualifiers and remove duplication.
Nesting classes is a good way to qualify types.
Instead of `JsonReaderToken` or `JsonReader.JsonToken`, prefer `JsonReader.Token`.
Class names also qualify static members.
Instead of `RailwayFactory.createRailway()`, prefer `RailwayFactory.create()`.
Instead of `CertificatePinner.DEFAULT_CERTIFICATE_PINNER`, prefer `CertificatePinner.DEFAULT`.
In many cases, descriptive qualifiers and nonredundant names eliminate the desire to use static imports for nested classes and static members.
However, there are exceptions. `Math.sqrt(double)` and `TypeName.LONG` read better imported as `sqrt(double)` and `LONG`.
Kotlin's package namespace works well for these cases. Move appropriate pure functions and constants to be top-level members in Kotlin.
In Kotlin code, instead of using utility classes (like `Math`), name the functions and constants appropriately and forgo class name qualifiers.
Besides being consistent and improving APIs, using qualifiers enhances implementation readability.
```
class AvatarView {
  // Use a nested class instead of an AvatarViewListener class in the package.
  interface Listener { void onClick() }

  // Listener, instead of fully qualified AvatarViewListener.
  bind(Listener listener) {}
}
```

Remove duplicate descriptors.
Consider a Java field:
~~~
Train internalTrain
~~~
The name repeats the visibility and the type. The name is an opportunity to give context to the held value, like `stationed`.

When the meaning of a generic type argument is ambiguous, add more data to the generic type argument name. Instead of `T`, prefer `ResponseT`. The `T` remains as a reminder that the type is a generic type argument and not a real type.

Prefer camel-cased acronyms. Capitalized acronyms in multiword phrases without spaces are hard to parse. Instead of `SOAnswer`, prefer `SoAnswer` or, avoiding the acronym, `StackOverflowAnswer`.

[Strict naming conventions are a liability.](https://publicobject.com/2016/01/20/strict-naming-conventions-are-a-liability/) Use naming conventions that give clues to the code's usage or origin.

Relatedly, retain the synthetic feel to generated code in its naming for internal and external usage.

These are naming styles and designs I found myself using, and I wrote them down here to state them explicitly in my head.
