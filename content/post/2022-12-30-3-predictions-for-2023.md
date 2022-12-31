---
title: "3 predictions for 2023"
date: 2022-12-30
tags: ["programming"]
---
What will happen next year?

## WebAssembly will replace Docker

What happened:

- Programmers pack their apps in Docker images together with the whole OS.
- This leads to
  - more security issues (if images are not frequently updated)
  - and waste of computing resources
    (to run an app whole OS in Docker image must be started).

It's sad that we use VMs like JVM or CLR which should work cross-platform.
But they don't so we rather pack the application together with the whole OS.

Solutions:

- Make existing VMs work cross-platform.
- Improve cross-compilation in existing compilers.
- If isolation is a concern compile to WebAssembly (Microsoft's Azure Kubernetes Service
  already supports WebAssembly System Interface node pools).

## Languages will implement stackful coroutines

What happened:

- Many languages implemented stackless coroutines.
- Many of these implementations are
  - hard to use
  - and partition functions into two groups synchronous and asynchronous
    which harms reusability and makes program maintenance harder.
    - Eg. if synchronous function wants to print a line, should we make it asynchronous?
      If its caller was synchronous should we make it asynchronous too?

Solutions:

- Implement stackful coroutines.
  - Go and Crystal already have them.
  - Java and OCaml will have them soon.
  - Rust's safety seems to currently not work well with stackful coroutines
    (eg. access to thread local storage).
  - .NET is experimenting with them (in branch
    [green-threads](https://github.com/dotnet/runtimelab/tree/feature/green-threads)).
- Interesting idea which doesn't solve the problem itself
  is to allow programmer to manually manipulate with stack frames -- like Zig.

## Reasonable GUI library will emerge

What happened:

- Lots of GUI libraries exist.
- These are either:
  - Written for single language without bindings to other languages (Flutter for Dart,
    Jetpack Compose for Kotlin, JavaScript libraries for JavaScript, MAUI for C#,
    Avalonia for C#, SwiftUI for Swift, ...).
  - Complex (Qt, Gtk).
  - Simplistic.
  - Immediate mode and having trouble with complex layouts (Dear ImGui, egui).
  - Not cross-platform (SwiftUI).

Currently I like SolidJS and egui.
Both lack Virtual DOM so they're simpler and have better performance.
Unfortunately SolidJS is bound to JavaScript and web browser and egui to Rust.

Solution:

- Create GUI library in low-level language using ideas from SolidJS.
- Create good documentation.
- Create bindings for high-level languages.
- Proper support for text shaping and editing would be nice
  (input method editors, bidirectional text, combining characters, ...).
