---
title: "Writing high-performance code for JVM in Kotlin"
date: 2020-03-08
tags: ["programming", "kotlin"]
---
Writing low latency, high througput, garbage-free system? Kotlin is an excellent choice.

## Inline functions

Guaranteed inlining is Kotlin's major advantage over Java when writing high-performance code:

* Generic inline functions specialize for primitive types so boxing is avoided.
* Lambdas given to inline functions are inlined as well so they're not allocated on the heap.

Inlining of lambdas has another interesting consequence:

* `return` used inside lambda is inlined as well so it will return from the enclosing function.

The first two points avoid garbage which consequently improves both latency and throughput. The third point promotes code reusability since if you want to stop early you don't have to write another higher-order function (or throw exception as practiced in Scala).

## `Array<T>` always contains references to objects

Array with primitive values can be constructed by `IntArray`, `BooleanArray`, etc. On the other hand `Array<Int>` is array of boxed integers repesented by `java.lang.Integer`.

## Type alises

Using specific types makes your program easier to understand. Eg. `Price` over `BigDecimal`. In Kotlin `Price` can be declared as type alias for `BigDecimal`. Unlike wrapper classes type aliases have no runtime costs.

## Inline classes

Type alias only introduces different name for existing type. If you really need a new distinct type create an inline class. Inline class is a wrapper class which Kotlin compiler tries to eliminate at compile time. The rules for elimination of inline classes are complex so if you need to avoid heap allocation altogether then avoid inline classes.

## Final by default

Classes and methods in Kotlin are final by default. This makes it easier for JVM to apply certain optimizations.

## Conclusion

When writing high performance code Kotlin has certainly something to offer. Unfortunately in 2020 JVM still lacks user-defined value types which became the cornerstone for high-performance, garbage-free systems in .NET. So we're waiting for Project Valhalla to bring this feature to JVM.
