---
layout: post
title:  Retiring Overly-specific Protocols
date:   2021-08-26 07:07:00 -0700
author: vanessa
tags: swift ios
category: software
---
<!-- cspell:ignore Quimby -->
When you start a project, and you try to plan for the future. Protocols help you do that &mdash; you use protocols to define what objects can do, what their interface is, and your vision is a neatly-scoped protocol-oriented future.

However, as your product goes, so do your protocols. One property here, another method there. And now you've got protocols that do _a lot of things_. So many things, that not everything in the protocol definition needs to be done by all things implementing that protocol. 

Now what do you do?

## The Right Answer™️, and Its Cost
There's an easy-to-say-yet-possibly-costly right answer. If your protocol has a method that _not all_ who conform to it need to implement, it shouldn't be in the protocol. Refactor that thing out. Either

- It doesn't need to be in a protocol, anywhere,
- It should be in its own protocol, and then only some things should inherit it
- Or it can be part of a protocol composition that combines things...

All of these are great, reasonable answers. 

All of these might require _considerable_ work, depending on the size of the project, and the number of things conforming to that protocol. If time or cost is a factor, what are the alternatives?

## A Simple Example.
Let's imagine a protocol for traffic lights. You define it like this:

```swift
protocol TrafficLight {
    var color: LightColor { get }
    
    mutating func turnRed()
    mutating func turnYellow()
    mutating func turnGreen()
}
```

Then, you spend years conforming to this protocol, and build your city. Until one day, Mayor Quimby decries that, to increase traffic flow, there will be no more green lights -- only red & yellow

![Lenny waiting for a red light to turn yellow, then flooring it](/assets/img/red-yellow-light.gif)

## Approach 1: Ignore it
By far, the easiest thing to do is to just ignore it.

```swift
struct FasterTrafficLight: TrafficLight {
    // ...
    mutating func turnGreen() {
        // Don't turn green anymore.
    }
}
```

While this meets the protocol, it also fails silently. This can lead to very hard-to find bugs. For instance, if the `TrafficLight` controller waited for the light to turn green before turning it red again, the light could **never** turn red, and you wouldn't know why.

Let's try something that gives us an error. 

## Approach 2: Crash
The quickest way to an error is a crash. Let's try that.

```swift
struct FasterTrafficLight: TrafficLight {
    // ...
    mutating func turnGreen() {
        fatalError("Faster traffic lights don't turn green")
    }
}
```

This works well: anytime someone tells this light -- which _shouldn't_ turn green -- to turn green, it will crash the program. We'll know immediately that someone is using this method in a way they're not supposed to. 

Unfortunately, so will users. Crashing is an effective way of stopping your program in an invalid state, but someone using the protocol won't know that this is an invalid method to use. You could use a different crash method that only works on debug builds, like `assertionFailure`, but you might still crash the app during normal usage.

## Approach 3: Annotating the method
Maybe we could annotate the method somehow, so we can catch misuse at compile time, not run time? For instance, what if we label the method as unavailable:

```swift
struct FasterTrafficLight: TrafficLight {
    // ...
    @available(*, unavailable)
    mutating func turnGreen() {
        // Don't turn green anymore.
    }
}
```

This... flat out doesn't work. With this method marked `unavailable`, it no longer conforms to the `TrafficLight` protocol.

What about `deprecated`?

```swift
struct FasterTrafficLight: TrafficLight {
    // ...
    @available(*, deprecated)
    mutating func turnGreen() {
        // Don't turn green anymore.
    }
}
```

This is better. It compiles, and if we use `FasterTrafficLight` directly, the compiler _will_ warn us.

```swift
var light = FasterTrafficLight()
light.turnGreen() // Warning: turnGreen() is deprecated
```

This is pretty good. It tells other developers that they shouldn't use this method, and if used directly, the compiler will warn you that you're about to do something wrong.

However, it's not exactly what we want. Anything that uses the protocol instead of the concrete type won't see a warning, even if the underlying type is using the deprecated method.

```swift
var light: TrafficLight = FasterTrafficLight()
light.turnGreen() // This is fine 🐶☕️🔥
```

## Conclusion
None of these are perfect solutions. None outperform removing `turnYellow` from the `TrafficLight` protocol and auditing its usage. But they're all significantly cheaper to implement.

Personally, I like the idea of both marking it `deprecated` and using `assertionFailure` to crash debug builds. That's the right level of annoying for me, but figure out what works best for your project and your team. And when you can, take the time to do that proper refactor.