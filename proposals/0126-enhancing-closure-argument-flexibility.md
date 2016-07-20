# Enhancing closure argument flexibility

* Proposal: TBD
* Authors: [Matthew Johnson](https://github.com/anandabits), [Erica Sadun](http://github.com/erica), [Rob Napier](http://github.com/rnapier)
* Status: TBD
* Review manager: TBD

## Introduction

This proposal loosens closure requirements to support developer
flexibility. It removes the `_ in` requirement that bypasses explicit
argument use and allow closures to use any, all, or none of the implicit
`$n` variables as needed or desired.

*The Swift-evolution thread about this topic can be found here: 
[Removing "_ in" from empty closures](http://thread.gmane.org/gmane.comp.lang.swift.evolution/17080)*

## Motivation

Swift closures that do not explicitly declare an internal parameter list
must reference all arguments using implicit `$n` shorthand names. If
they do not, Swift complains that the contextual type for the closure
argument "expects `n` arguments, which cannot be implicitly ignored."
This requirement diminishes the efficacy of Swift's `$n` syntactic
sugar. Eliminating the requirement means:

* `{}` becomes a valid 'noop' closure in any context requiring a `Void`-returning closure. 
* Implementations can discard unnecessary code cruft and streamline their minimum implementation from `{ _(, _)* in }` to `{}`.
* `{ expression }` becomes a valid closure in any context requiring a return value. The expression can offer a simple expression or literal, such as `{ 42 }`.
* The closure can mention *some* of its parameters without having to mention *all* of its parameters.

Furthermore, the current approach is inconsistent in how it treats `$0`
versus other parameters. `$0` is treated as the entire tuple, while `$1`
is treated as the second element of the tuple. For example, given the
closure:

    typealias Closure = (Int, String) -> Any
    let f: Closure = { $0 }
    f(1, "Test")          // (.0 1, .1 "Test")
    let g: Closure = { $1 }
    g(1, "Test")         // "Test"

This proposal changes `$0` to be of type `Int` (the first parameter) in this case.

## Detailed Design

Under this proposal, parameterized closures types will automatically
promote closures that mention a possibly empty subset of implicit
arguments. All the following examples will be valid and compile without
error or warning:

```
let _: () -> Void = {}
let _: (Int) -> Void = {} 
let _: (Int, Int) -> Int = { 5 }
let _: (Int, Int) -> Int = { $0 }
let _: (Int, Int) -> Int = { $1 }
```

In the first two examples, the empty closure literal `{}` will
autopromote to satisfy any void-returning closure type `(T...) -> Void`.

This proposal only applies to closures that would be eligible for `$N`
shorthand names. Closures that name some parameters must still provide
placeholders for all of them. For example:

Given a closure of type `(Int, String) -> Int`, the following closures would be valid:

    { 1 }
    { $0 }
    { x, y in x }
    { x, _ in x }
    { _, _ in 1 }
    
The following closure would not be valid:

    { x in x }

```swift
// Current
doThing(withCompletion: { _ in })
let x: (T) -> Void = { _ in }

// Proposed
doThing(withCompletion: {})
let x: (T) -> Void = {}
```

In the remaining examples, the closure will support the return of any
expression, whether or not it mentions any or all of the implicit
arguments.

## Impact on Existing Code

This proposal modifies the treatment of $0 in multi-parameter closures.
Closures that rely on $0 being a tuple of all of the parameters would
need to be modified. This may be resolvable with a fixit. Other closures
will continue to compile and run as they did prior to adoption. It
offers positive improvements for new code added after adoptions. We
believe it would be beneficial for Xcode to scan for `{_(, _)* in}`
patterns during migration and offer fixits to update this code to `{}`.

## Alternatives Considered

We considered and discarded a policy where Swift encourages the use of
optional closures in place of simplifying the `{}` case. This approach
does not scale to non-Void cases. It does not align with Cocoa APIs,
where most completion handlers are not optional.

## Related Proposals

* [SE-0029: Remove Implicit Tuple Splat Behavior from Function Applications](https://github.com/apple/swift-evolution/blob/master/proposals/0029-remove-implicit-tuple-splat.md) removed the behavior that allowed `$0` to represent either the first argument *or* a tuple of all arguments.
