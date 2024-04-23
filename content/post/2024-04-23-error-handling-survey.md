---
title: "Error handling survey"
date: 2024-04-22
tags: ["programming", "error-handling", "odin", "go", "zig", "rust", "fsharp", "ocaml"]
---

Errors are everywhere. Almost every operation can result in an error.
Basic arithmetic on integers may result in overflow or division by zero.
A procedure call may result in a stack overflow. Simple string concatenation
may result in an out of heap memory error. Let's evaluate how programming
languages Odin, Go, Zig, Rust, F# and OCaml deal with errors.

## Kill the program

There are several basic strategies for dealing with errors.
The simplest one is to kill the whole program when an error happens.
This is especially helpful when a program can't recover from the error
and killing it prevents it from continuing in a corrupted state.

For example, in Java it's very likely that `OutOfMemoryError` results in a corrupted state.
The reason is that almost every operation in Java allocates memory.
So almost every operation may result in `OutOfMemoryError`.
And it's very hard to ensure a consistent state or rollback changes into a consistent state
while almost every operation may fail (even when doing the rollback).
So catching `OutOfMemoryError` in Java with `catch (Throwable t)` clause is almost certainly a bug.
Killing the whole program should be used instead.

There are languages like Zig which are very explicit about heap memory allocation
and where programs can work correctly even when running out of memory, but Java is not one of them.

## Throwing an exception or panicking

An alternative to killing the whole program is to throw an exception or panic.
Both exceptions and panics are technically the same mechanism:
Call stack unwinding, during which resources are being cleaned up until
an exception or panic handler catches the error.
If there is no handler, then the thread or the program is killed.

Languages like Go and Rust have panics.
Go kills the whole program when panic is not caught by `recover` function.
On the other hand, Rust kills only the thread where panic occurred.
Solely killing the thread may lead to a program being left in a corrupted state.
Many other languages have exceptions.

The crucial property of exceptions and panics is that they bubble up automatically.
You don't have to write special code to do it.

## Returning an error description instead of a normal result

Another way to deal with errors is to return an error description instead of a normal result.
This means that the result type is a union, which contains either a normal result
or an error but not both.
This is common in languages with discriminated unions - eg. OCaml, F# and also Rust.
Many of these languages use the special type `Result<T, E>`
which either contains a normal result `T` or an error description `E`.

The main difference from exceptions is that you have to do the bubbling manually
if you want it. To simplify this Rust has a special operator `?`
which on success converts `Result<T, E>` to `T` and on failure it returns the whole result from
the current function. This mechanism is extensible and works on types implementing trait `Try`.
For example, if `parse` can fail, then you can write `let i: i32 = "123".parse()?` to either
get an integer or exit the function with the error from `parse`.

Zig has a similar mechanism called `try` which is completely different from `try` known from Java,
C#, F# C++ and Python. Instead it works as in Rust's `?`.
With it you can write `const i: i32 = try parse("123")`.
If parse returns an error, then `try` will exit the current function with that error.
Otherwise it will unwrap the normal result from the error union. `try` in Zig is actually
a shorthand for catching the error and returning it from the function:

```zig
const i: i32 = parse("123") catch |e| return e;
```

The problem with error unions in Zig is that the errors can't carry any payload.
For example, if a function works with several files and its return type
is the error union `FileOpenError!void` where

```zig
const FileOpenError = error {
    AccessDenied,
    FileNotFound,
};
```

then in case of an error you can't know which file caused the error.

The problem with this approach is when callers are allowed to ignore results with errors.
Then it's easy to ignore an error and continue as if everything is alright.
To remedy this `Result` in Rust is annotated with `#[must_use]` so it can't be ignored.
Compilers of OCaml and F# warn a user whenever a result of non-unit type is ignored.
Zig compiler is even stricter and forces a user to use all non-void results
or explicitly ignore them.

## Returning both an error description and a normal result

The less common approach is to always return both a normal result and an error description.
This is implemented in Go and Odin. The programmer must then be careful
and check that no error occurred before using the normal result.

There's no special facility for bubbling in Go.
On the other hand Odin provides `or_return` keyword where `res := f() or_return`
is a shorthand for

```odin
res, err := f()
if err != nil {
    return err
}
```

Unfortunately, Go allows users to ignore return values.
Odin by default does the same. Additionally Odin has the annotation
`@(require_results)` which forces users of annotated functions to capture all return values.

In Go either all return values must be captured or none.
Odin by default does the same. Unfortunately, it has another annotation `#optional_allocator_error`
which allows users to ignore `Allocator_Error`.

Examples of functions annotated by `#optional_allocator_error`
are the built-in functions `make` and `append`.
When they fail, they return an error description of type `Allocator_Error`
which is due to this annotation ignored in many code bases.
On the plus side, calls to `make` and `append` are less cluttered.
On the negative side, I believe it makes writing reliable software in Odin much harder.

## Errno

Instead of returning an error description and a normal result
a function returns a normal result and sets an error description in some global
or thread-local variable usually called errno. The caller then needs to check this variable
to determine if an error happened.

Unfortunately, nothing forces the caller to check errno.
So it's easy to continue while in a corrupted state.

## Pretend that no error happened

This strategy is frequently used for arithmetic overflow.
Previously mentioned strategies except for exceptions and panics are impractical
for arithmetic overflow. For example, suppose that operators for basic arithmetic
return unions and the user must check the result of every operation whether it succeeded or not.
That would clutter the code and slow down the computations.

Instead Odin and Go solve integer overflow by wrapping around.
This is dangerous for counters and sizes and may be quite hard to test.
For example, how would you test how your data structure behaves when its size is close
to the maximal integer when you don't have so much memory?
So, in my opinion, it would be much better to throw an exception or panic by default.
For other behaviors like wrap around or saturation it's better to have different operators.

## Undefined behavior

The last approach to dealing with errors is to make them an undefined behavior.
This basically means that the programmer is responsible for avoiding them
and if they fail then the program may go crazy.

Undefined behaviors simplify the duty of the compiler. For example, if integer overflow
is undefined, the compiler may assume that it won't happen.
If it actually happens, then it's not defined what the program shall do,
so the compiler doesn't have to concentrate on this case.

This strategy is used by C, C++ and Zig and unfortunately has led to many bugs.
Another problem is that different compilers or even different versions of the same compiler
may behave differently. So, for example, if a program was working in one version, that
doesn't mean it will work in another. Or it may behave differently.

## Knowing your errors

Another important criterion is whether the programmer knows
or has a chance to know which errors could happen.
For example, none of the languages with a garbage collector and exceptions bothers
to document which operations may result in an out of memory exception.
And this is a problem. For example, if you want to change a data structure
transactionally, you need to know which operations allocate heap memory and may fail.
If you know it, you can do the allocations first and only if they succeed do the changes
in the data structure.

Another example is that you know which errors could happen but you don't know when.

The good thing about exclusively returning an error description is that
the return type documents all possible errors. With exceptions it's harder.
Java has checked exceptions which ensure that methods are annotated
if they throw certain exceptions.
The problem with this is that it doesn't work for all exceptions, so it's useless in practice.
Some non-mainstream languages have type systems that describe side effects, including exceptions.
But this is rare.

## Propagating your errors

The advantage of exceptions and panics is that they automatically bubble up.
On the other hand, returning an error description
usually requires converting errors of one type to errors of a different type.
For example, if a user writes a function that calls functions `f` and `g`
where `f` may fail with `ErrorF` and `g` with `ErrorG`, then
their function may fail with `ErrorF` or `ErrorG`.

Rust users solve this problem by writing special crates for error handling
like `thiserror`, `anyhow` and `eyre`. It's sad that Rust with
its standard library doesn't provide anything reasonable out of the box for
such ubiquitous thing as error handling.

In OCaml if errors are represented by polymorphic variants, then they can be easily
combined to the new polymorphic variant `[error_f | error_g]`.
Additionally, OCaml can infer this type by itself.

In Odin the user needs to create a union type

```odin
Both_Errors :: union {
    ErrorF,
    ErrorG,
}
```

Both `ErrorF` and `ErrorG` can be lifted to it by `or_return`.

F# doesn't have any special facility. In Go it's common that errors
implement `error` interface. So functions in `Go` that may fail usually return
`error` and so it's impossible to know for sure which errors may occur.

In Zig we can combine error sets with `||` operator. So the user needs to return
the error of type `(ErrorF || ErrorG)` or its superset
and in many situations Zig compiler can infer this.
Since members of the error set `ErrorF` also belong to `(ErrorF || ErrorG)`
no conversion or lifting is needed.

## Summary

Now we have covered different approaches for dealing with errors.
I believe that if you want to write reliable software,
you must not ignore errors. So languages or libraries where that may happen by accident
are not suitable.

It's also very helpful if you know which errors may happen and if this
information is checked by a compiler, so you can rely on it
and decide which errors handle and which let to bubble up.

Technically, it's hard to rate languages based on this.
Because many languages allow multiple strategies and sometimes
the problem is just bad choices in libraries.

Generally, I find Zig to be the best if you stick to debug or release safe
build modes which panic on undefined behaviors.
In Zig compiler won't let you accidentally ignore errors and you also know which errors
may happen. The only downside is that errors can't carry any context.

I have spent more than 10 years of programming in F#. It's the language that offers
multiple ways of error handling but none of them is superb.
Good thing is that you can't ignore exceptions but the bad thing is
that you don't know what can throw or which exceptions expect.
You even don't easily know which language constructs allocate memory on the heap.
Thus, for example, ensuring that a code leaves a data structure in a consistent state
is extremely hard because you don't know what can throw `OutOfMemoryException` and what is safe.

By default, operators in F# wrap around on overflow.
Fortunately in F# you can open module `FSharp.Core.Operators.Checked`
to get operators that throw an exception on overflow.
Integers in OCaml wrap around in case of overflow.

Overflows in Rust cause panic in debug mode and wrap around in release mode.
This is unfortunate in a language that focuses on safety.
Another unfortunate aspect in Rust is that error handling isn't superb
out of the box, and you need to install third pardy dependencies.

Go and Odin don't consider integer overflow as an error - they instead wrap around.
Go kills the program when it runs out of memory, which is a good thing.
In Odin you can accidentally ignore `Allocator_Error` and continue in a corrupted state.
These problems usually don't appear during development when using the default allocator
and may be hard to replicate.
