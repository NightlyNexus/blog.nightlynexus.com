---
layout:     post
title:      Public is the Only Worthwhile Visibility Modifier
date:       2018-09-12 23:45:00
summary:    Private is code pollution.
categories: Android
---

`public` is the only worthwhile visibility qualifier. Code is either API or implementation details. Everything should default to an "internal" visibility, where "internal" code is implementation details (of a package, build module, or whatever the code is packaged in that ships functionality via a public API). `private` is a noisy modifier. Having code encapsulated in a class offers little benefit over having code encapsulated in a package/build module/etc. The detriment of the noise of the `private` modifier outweighs the benefit of having implementation details within implementation details via the `private` modifier. The `protected` modifier for subclasses is nearly useless as well. Similar to `private`, if `protected` is used for internal code, it is redundant with "internal" visibility. If `protected` is used to expose functionality to subclasses of a public class in an API, it is redundant with the `public` visibility.

The only hypothetical case for `protected` that I know of is as follows:
- Code C consumes Library B.
- Library B consumes Library A.
- Library A has a Class A in its API designed for extension with an API with `protected` visibility.
- Library B has a Class B that is a subclass of Class A.
- Class B uses the protected API of Class A.
- Class B is final/not designed for extension.
- Class B is still public API for consumption by Code C.
- The protected API of Class A in Library A is now an implementation detail of Class B in Library B.
- Code C now uses Class B and has no access to the protected API of Class A via Class B, whereas, if the `protected` visibility had been `public` instead, Code C could have used the protected API of Class B, despite Library B's intentions.

In Library A:
~~~
public abstract class ClassA {
  protected abstract void run(); // Public API for consumers of Library A.
}
~~~
In Library B:
~~~
public final class ClassB extends ClassA {
  @Override protected void run() { // Implementation detail of Library B.
  }
}
~~~
In Code C:
~~~
final class ClassC extends ClassA {
  @Override protected void run() {
  }
}

void main() {
  ClassC codeCImplementation = new ClassC();
  codeCImplementation.run();

  ClassB fromLibraryB = new ClassB();
  // fromLibraryB.run(); Will not compile. This is an implementation detail of Library B.
}
~~~

I have never seen this case in my experience, and, if `protected` were not a visibility modifier, Library B's API could work by exposing a different class besides ClassB.

Java gets visibility just about right yet adds the needless `private` and `protected` modifiers. Do not use them. Kotlin expands on package visibility with internal visibility which serves the same purpose and allows for a little more code organization for implementation details. But, Kotlin goes wildly wrong by making everything public API by default. With this default, all code has to be littered with the `internal` modifier. If we already have to litter our code with a visibility modifier, we might as well use `private` when we can. As argued above, `private` and `protected` buy us little. But, `internal` is just as noisy as `private` or `protected` on all implementation code, so we might as well take the marginal benefit. Kotlin's visibility is unfortunate and distracting.

Everything should default to internal implementation. Public specifies public API. All other modifiers are noise.
