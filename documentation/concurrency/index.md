---
layout: page
date: 2024-03-03 12:00:00
title: Migrating to Complete Concurrency Checking
---

Concurrent programs are notoriously difficult to write correctly and understand. A major factor in the difficulty of concurrent programming is the unsafety caused by data races. A _data race_ occurs when one thread accesses memory while the same memory is being mutated by another thread. Data races are exceptionally difficult to reproduce, debug, and fix because runtime failures are nondeterministic; the program may only fail under certain conditions that are impossible to replicate locally.

The Swift 6 langauge mode, which enables complete concurrency checking by default, solves these problems by preventing data races at compile time. This document will walk you through the concepts used throughout Swift's data-race safety model, and the steps to prepare your code for migrating to the Swift 6 language mode to eliminate data races from your code.

## Concepts

Swift provides a variety of language-level features to write data-race-free concurrent code. For an overview of `async`/`await`, structured concurrency, and actors, please see the [Concurrency section of The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency). This guide will specifically cover language concepts to write code that is correct under full data isolation with complete concurrency checking.

### Data isolation

Swift concurrency prevents data races by guaranteeing mutually exclusive access to mutable state through _data isolation_. Mutable state is protected by an _isolation domain_ associated with a task or an actor. Functions that access the state must run in that task or actor. Functions in the same isolation domain cannot run concurrently with each other, but they can run concurrently with functions in other isolation domains.

Mutable state can only be accessed from one isolation domain at a time. You can pass mutable state from one isolation domain to another, but you can never access that state concurrently from different isolation domains at once without synchronization. This guarantee is validated by the compiler.

Tasks are the basic unit of concurrency; multiple tasks may run concurrently with each other, but each individual task only executes one function at a time. Tasks can either be isolated to an actor, or they can be non-isolated. All tasks have access to their local state, task-local values, and global-nonisolated state, and isolated tasks also have access to actor-isolated state.

### Isolation domains

All functions and variables have a static isolation domain that is understood by the compiler. The isolation of a declaration can be:

1. Non-isolated
2. Isolated to an actor value
3. Isolated to a global actor

When a declaration is isolated to an actor or global actor, it means the declaration can only be called or accessed from code running on that actor. Non-isolated declarations are not isolated to a specific actor, and therefore cannot acccess any actor-isolated state. Synchronous functions and variables that are non-isolated can be called or accessed from any isolation domain. Non-isolated async functions always run on the generic executor that is not associated with any actor. When a non-isolated async function is called from an actor-isolated context, the call crosses an isolation boundary. When called from another non-isolated context, the call does not cross an isolation boundary.

By default, functions and properties of non-isolated types are also non-isolated. A function or variable can also be explicitly non-isolated with the `nonisolated` keyword. A function or property that is `nonisolated` is safe to access from any isolation domain because it cannot access any actor-isolated state in its implementation.

All stored instance properties of an actor are isolated to the enclosing actor instance. Functions are isolated to an actor instance if they accept an `isolated` parameter, including the `self` parameter to actor instance methods:

```swift
actor MyActor {
  var count: 0

  func increment() {
    count += 1
  }
}

func increment(on myActor: isolated MyActor) {
  myActor.count += 1
}
```

In the above code, `MyActor.increment` is isolated to `self`, and `increment(on:)` is isolated to the `myActor` parameter value, so both functions have access to the isolated state `count`.

Functions and variables with an explicit global actor attribute, such as `@MainActor`, are isolated to that global actor. If no isolation is specified, global actor isolation is inferred in the following cases:

* The declaration is a member of a global-actor-isolated type as described below
* The declaration is written in a global-actor-isolated extension
* The declaration satisfies a global-actor-isolated protocol requirement, and the declaration is in the same scope that the conformance is declared in
* The declaration is a function override of a global-actor-isolated superclass method

An entire type can be isolated to a global actor. When a type is isolated to a global actor, all properties and functions inside the type are isolated to that global actor unless a different isolation is specified, such as `nonisolated`. A type is isolated to a global actor if it has an explicit global actor attribute, or the global actor is inferred under one of the following conditions:

* The type conforms to a protocol that has a global actor attribute, and the conformance is not stated in an extension
* The type is class that inherits from a global-actor-isolated superclass

Global variables in a `main.swift` file are implicit isolated to `@MainActor`, and `static func main` is implicitly isolated to `@MainActor`.

#### Suspension points

A task can switch between isolation domains when a function in one isolation domain calls a function in a different isolation domain. When a call crosses an isolation boundary, that call must be made asynchronously, because the destination isolation domain might be busy running other tasks. In that case, the task will be suspended until the destination isolation domain is free to run the function. A suspension point does not block, meaning the current isolation domain (and the thread it is currently running on) are freed up to perform other work. The Swift concurrency runtime expects code to never block on future work, allowing the system to always make forward progress, which eliminates a common source of deadlocks in concurrent code.

Potential suspension points are marked in source code with the `await` keyword. The `await` keyword indicates that the call might suspend at runtime; `await` does not force a suspension point, and the function being called might only suspend under certain dynamic conditions, so it's possible that a call marked with `await` doesn't actually suspend. In any case, explicitly marking potential suspension points is important in concurrent code because suspensions indicate the end of a critical section. Because the current isolation domain is freed up to perform other work, actor-isolated state may change across a suspension point. As such, your critical sections should always be written in synchronous code.

### Passing data across isolation boundaries

Values can be passed across isolation boundaries when there is no potential for concurrent access to shared mutable state. Some values are always safe to pass across isolation boundaries because the type of the value is inherently thread-safe, and other values may be safe to pass across isolation boundaries because the original isolation domain does not access the value's mutable state after the point of transfer.

#### `Sendable` types

In some cases, all values of a particular type are safe to pass across isolation boundaries because thread safety is a property of the type itself. This thread-safe property of types is represented by a conformance to the `Sendable` protocol. When you see a conformance to `Sendable` in documentation, it means the given type is thread safe, and values of the type can be shared across arbitrary isolation domains without introducing a risk of data races.

Swift encourages using value types because they are naturally safe from data races. When you use value types, different parts of your program can't have shared references to the same value. When you pass an instance of a value type to a function, the function has its own independent copy of that value. Value types in Swift are implicitly `Sendable` within the module when their stored properties are `Sendable`, because value semantics guarantees the absence of shared mutable state. Note that public value types must explicitly state a conformance to `Sendable`, because it's part of the public API contract.

Actors and global-actor-isolated types are also implicitly `Sendable`, because their mutable state is protected by actor isolation. It's safe to pass an actor or actor-isolated type around, but code must switch back to the actor in order to access its isolated state. `nonisolated` properties and functions may still be accessed from outside the actor, because `nonisolated` functions cannot access actor-isolated state:

```swift
struct User: Sendable { ... }

@MainActor class MyModel {
  nonisolated let id: String
  private var users: [User] = []

  nonisolated func printID() {
    print(id)
  }

  func add(user: User) {
    users.append(user)
  }
}
```

In the above code `MyModel` conforms to `Sendable` because all of its mutable state is isolated to the main actor. An instance of `MyModel` can be used from outside the main actor, but only the `id` property or the `printID()` method can be used. A call to `add(user)` from off the main actor must be done asynchronously in order to switch to the main actor to append a value to the isolated `users` property:

```swift
let myModel = MyModel(id: "example")
Task.detached {
  myModel.printID()
  await myModel.add(user: User(...))
}
```

Reference types are only `Sendable` if they do not have any mutable state, or if the type implements its own synchronization. `Sendable` is never inferred for class types. The compiler can only validate the implementation of final classes; it's safe for a final class to be `Sendable` as long as its stored properties are either immutable and `Sendable`, or isolated to a global actor.

It is possible to implement thread-safety using synchronization primitives that the compiler cannot reason about, such as through OS-specific primitives or when working with thread-safe types implemented in C/C++/Objective-C. Such types may be marked as conforming to `@unchecked Sendable` to tell the compiler that the type is thread-safe. The compiler will not perform any checking on an `@unchecked Sendable` type, so this opt-out must be used carefully.


## Enabling strict concurrency checking

Data-race safety in the Swift 6 language mode is designed for incremental migration. You can address data-race safety issues in your projects module-by-module, and you can enable the compiler's actor isolation and `Sendable` checking as warnings in the Swift 5 language mode, allowing you to assess your progress toward eliminating data races before turning on the Swift 6 language mode.

Complete data-race safety checking can be enabled as warnings in the Swift 5 language mode using the `-strict-concurrency` compiler flag.

### Using the Swift compiler

To enable complete concurrency checking when running `swift` or `swiftc` directly at the command line, pass `-strict-concurrency=complete`:

```
~ swift -strict-concurrency=complete main.swift
```

### Using SwiftPM

#### In a SwiftPM command-line invocation

`-strict-concurrency=complete` can be passed in a Swift package manager command-line invocation using the `-Xswiftc` flag:

```
~ swift build -Xswiftc -strict-concurrency=complete
~ swift test -Xswiftc -strict-concurrency=complete
```

This can be useful to gauge the amount of concurrency warnings before adding the flag permanently in the package manifest as described in the following section.

#### In a SwiftPM package manifest

To enable complete concurrency checking for a target in a Swift package using Swift 5.9 or Swift 5.10 tools, use [`SwiftSetting.enableExperimentalFeature`](https://developer.apple.com/documentation/packagedescription/swiftsetting/enableexperimentalfeature(_:_:)) in the Swift settings for the given target:

```swift
.target(
  name: "MyTarget",
  swiftSettings: [
    .enableExperimentalFeature("StrictConcurrency")
  ]
)
```

When using Swift 6.0 tools or later, use [`SwiftSetting.enableUpcomingFeature`](https://developer.apple.com/documentation/packagedescription/swiftsetting/enableupcomingfeature(_:_:)) in the Swift settings for the given target:

```swift
.target(
  name: "MyTarget",
  swiftSettings: [
    .enableUpcomingFeature("StrictConcurrency")
  ]
)
```

### Using Xcode

To enable complete concurrency checking in an Xcode project, set the "Strict Concurrency Checking" setting to "Complete" in the Xcode build settings. Alternatively, you can set `SWIFT_STRICT_CONCURRENCY` to `complete` in an xcconfig file:

```
// In a Settings.xcconfig

SWIFT_STRICT_CONCURRENCY = complete;
```

## Addressing data-race safety issues

### The approach

The recommended approach for addressing data-race safety issues in your project is in two phases:

1. First, express what is true in your code today using the annotations provided in the language.
2. Once all warnings are resolved, then look for opportunity to refactor parts of your code to take greater advantage of data-race safety.

Enabling complete concurrency checking in a module can yield many data-race safety issues reported by the compiler. Resist the urge to refactor your code to address these issues! It's beneficial to minimize the amount of change necessary to enable complete concurrency checking, and use any unsafe opt-outs you applied as an indication of follow-on refactoring opportunities to introduce a safer isolation mechanism such as an actor.

For example, if all code in your project runs on the main actor today, you should migrate to complete concurrency checking primarily by annotating code with `@MainActor`. Once complete concurrency checking is enabled, you can use those annotations to identify places where code is running on the main actor today, but doesn't need to be, so you can strategically introduce more concurrency.

### Categories of errors

There are a few different categories of data-race safety errors that you'll encounter when migrating to complete concurrency checking:

1. Concurrent access on argument values, result values, and closure captures
2. Unsafe global and static variables
3. Isolated method satisfying a non-isolated protocol requirement, or overriding a non-isolated superclass method
4. Calling an isolated method in a synchronous non-isolated context, or without `await`
5. Isolated default stored property values in a non-isolated context

