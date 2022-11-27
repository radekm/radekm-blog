---
title: "Caller info attributes in F# with currying"
date: 2021-02-27
tags: ["programming", "fsharp"]
---
How to use `CallerMemberName`, `CallerFilePath` and `CallerLineNumber`
in F# together with currying.

## Solution

Let's start with a solution. Instead of a module with functions
we declare a class with static methods:

```fsharp
type Mod() =
    static member noArg
        ( [<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int
        ) : string =
        sprintf "Line %d" line

    static member oneArg
        ( x : string,
          [<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int
        ) : string =
        sprintf "Line %d; input %s" line x

    static member twoArgs
        ( x : string,
          [<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int
        ) : string -> string = fun y ->
        sprintf "Line %d; inputs %s, %s" line x y

    static member threeArgs
        ( x : string,
          [<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int
        ) : string -> string -> string = fun y z ->
        sprintf "Line %d, inputs %s, %s, %s" line x y z
```

Hopefully it's obvious how to create functions which take more arguments.
We can test that it works as expected:

```fsharp
let _ = printfn "%s" (Mod.noArg ())
let _ = printfn "%s" (Mod.oneArg "hi")
let _ = printfn "%s" (Mod.twoArgs "hello" "world")
let _ = printfn "%s" (Mod.threeArgs "hello" "world" "!")
let _ = "works" |> Mod.twoArgs "piping" |> printfn "%s"
let _ = "bye" |> Mod.oneArg |> printfn "%s"
```

Note that the caller info attributes are recorded
when a static method is converted to a function:

```fsharp
let f = Mod.noArg
let _ = f () |> printfn "%s" // Prints line number of the previous line
                             // even though `noArg` was not called there.
```

Also note that normal methods can be used instead of static methods.
Readability can be improved by using F# optional parameters
instead of C# optional parameters:

```fsharp
type AnotherMod() =
    static member twoArgs
        ( x : string,
          [<CallerLineNumber>] ?line : int
        ) : string -> string = fun y ->
        sprintf "Line %d, inputs %s, %s" line.Value x y
```

A small drawback is that the parameter `line` has the type `int option`
instead of just `int` which makes this more readable alternative slightly less performant.


## Non-solutions

I would rather use functions instead of methods:

```fsharp
let wrong1
    ( x : int,
      [<CallerLineNumber>] line : int ) = x
let wrong2
    ( x : int,
      [<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int ) = x
let wrong3
    (x : int)
    ([<CallerLineNumber; Optional; DefaultParameterValue(0)>] line : int) = x
```

But these don't work since the compiler is not willing to fill `line` parameter.
The page
[Caller information](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/caller-information)
from F# language reference says that the caller info attributes work with methods,
and it doesn't mention functions at all. So I assume it's not possible to use functions.

Another thing is that we can't use optional arguments with curried methods.
For example `DoesNotCompileMod` class

```fsharp
type DoesNotCompileMod() =
    static member oneArg
        (x : string)
        ([<CallerLineNumber>] ?line : int) =
        sprintf "Line %d, inputs %s" line.Value x
```

really doesn't compile:

> [FS0440] Methods with curried arguments cannot declare 'out', 'ParamArray', 'optional',
> 'ReflectedDefinition', 'byref', 'CallerLineNumber', 'CallerMemberName', or 'CallerFilePath' arguments
