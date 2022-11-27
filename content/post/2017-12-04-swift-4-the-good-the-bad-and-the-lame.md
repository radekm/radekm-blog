---
title: "Swift 4: the Good, the Bad and the Lame"
date: 2017-12-04
tags: ["programming", "fsharp", "kotlin", "swift"]
---
How does Swift 4 stand against other popular languages? Should you use Kotlin or F# instead?

## Good: Retroactive modeling

Very nice feature of Swift is that you can make a type to conform to a protocol even if you don't have access to the source code of the type.

This is not possible with interfaces in Kotlin or F#.

## Good: Tuples with named fields

Another very good thing about Swift is the possibility to name fields of tuples. This is especially useful when returning multiple values from functions:

```swift
func minMax(arr: [Double]) -> (min: Double, max: Double) {
    // ....
}
```

Kotlin and F# don't support this. Instead you have to create a new type
and assign it a name. Eg.:

```fsharp
type MinMaxResult = { Min: double; Max: double }

let minMax arr =
    // ....
```

## Bad: Standard library is lacking

Let's compare two programs which use regular expression to extract items from the string `str`. The first one is in Swift 4:

```swift
let regex = try! NSRegularExpression(pattern: "\\* ([a-z]+)(?: \\(([a-z]+)\\))?")
let str = """
    * first
    * second (blah)
"""

let matches = regex.matches(in: str, range: NSRange(str.startIndex..., in: str))

let nsStr = str as NSString
for m in matches {
    let title = nsStr.substring(with: m.range(at: 1))
    if m.range(at: 2).location != NSNotFound {
        let desc = nsStr.substring(with: m.range(at: 2))
        print("Item '\(title)' with description '\(desc)'")
    } else {
        print("Item '\(title)' without description")
    }
}
```

The second one is in F#:

```fsharp
let regex = Regex("\\* ([a-z]+)(?: \\(([a-z]+)\\))?")
let str = "
    * first
    * second (blah)
"

let matches = regex.Matches str

for m in matches do
    let title = m.Groups.[1].Value
    if m.Groups.[2].Success then
        let desc = m.Groups.[2].Value
        printfn "Item '%s' with description '%s'" title desc
    else
        printfn "Item '%s' without description" title
```

Although the programs do exactly the same thing the Swift program is longer.

One problem with Swift is that some classes exist in two variants. For example `String` and `NSString` or `Range` and `NSRange`. Another problem is that Swift is missing some very useful properties so instead of

```fsharp
m.Groups.[2].Success
```

you have to write

```swift
m.range(at: 2).location != NSNotFound
```

Similar problem can be shown in other situations like downloading data from the web, processing XML, working with files, etc.

## Bad: No async support

Many modern programming languages have special support for *coroutines*. The main motivation is to simplify writing async code. With Swift 4 you have to manually split your async code into callbacks while Kotlin and F# compilers will do it for you.

## Bad: Reference counting

The main point of garbage collection is to promote *code reusability* by lifting the duty to care about memory. With reference counting in Swift 4 you have to care about memory because you have to prevent reference cycles.

On top of that reference counting is usually *much slower* than other forms of garbage collection if you have enough free memory. Which is usually the case for most devices today. On the other hand having more memory won't improve the performance of reference counting. You can see for yourself that some Swift programs in The Computer Language Benchmarks Game try to avoid reference counting by using `UnsafeMutablePointer`.

Memory management in Kotlin and F# is easier to use and it's also usually faster.

## Lame: Very slow compilation

Unfortunately Swift 4 can't compile simple functions like this one:

```swift
func sanitizeString(s: String) -> String {
    return s.filter { (c) in
        Character("A")...Character("Z") ~= c ||
        Character("a")...Character("z") ~= c ||
        Character("0")...Character("9") ~= c ||
        " -,".contains(c)
    }
}
```

The code is correct but too complex for the Swift type checker. Here is the error message:

> Expression was too complex to be solved in reasonable time; consider breaking up the expression into distinct sub-expressions

## Conclusion

Swift 4 has some really nice features but they don't cut it. You can write faster, shorter and more elegant code in Kotlin or F#. With Xamarin you can use F# to develop apps for Apple devices. And hopefully Kotlin/Native will soon bring Kotlin to iOS.
