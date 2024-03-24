---
layout: post
title:  The Golden Rules of `weak self`
date:   2022-04-10 11:45:00 -0700
tags: swift ios
category: software
---
Capturing `self` in a closure is a common thing to do in Swift, and one that hides a lot of nuance. Do you _need_ to make it `weak` to avoid a reference cycle? Is making it `weak` all the time a problem?

> This [past week's iOS Dev Weekly](https://iosdevweekly.com/issues/553?#start) featured [an opinionated article](https://benoitpasquier.com/weak-self-story-memory-management-closure-swift/) by [Benoit Pasquier](https://twitter.com/BenoitPasquier_) by  on capturing `self` in closures. This article will contradict it. And that's okay! Take all this advice with a grain of salt, understand the tradeoffs, and choose the techniques that suit you best.

Okay, let's dive in.

## Three Golden Rules
Reasoning about retain cycles is hard. When I teach folks about using `weak self` (or capture lists 😀) to avoid memory leaks, I introduce three golden rules:

1. A strongly-retained `self` isn't always a retain cycle
1. A weakly retained `self` will **never** be a retain cycle
1. Upgrade `self` to strong at the top of closures to avoid weird behavior

Let's see these rules in action.

## A Retain Cycle Example
A retain cycle is when an object retains itself. Here, a child class retains a closure referring to its parent class, causing a retain cycle:

```swift
class Parent {
    let child = Child()
    var didChildPlay = false
    func playChildLater() {
        child.playLater {
            self.didChildPlay = true
        }
    }
}

class Child {
    var finishedPlaying: () -> Void = {}
    func playLater(completion: @escaping () -> Void) {
        finishedPlaying = completion
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.2) {
            // Play! ⚽️🏀🥏
            completion()
        }
    }
}

let parent = Parent()
parent.playChildLater()
// `parent` is no longer used, but not recycled.
```

We can see that retain cycle represented in this diagram:

![A diagram showing a parent class retaining its child, that child retaining its closure, and a closure retaining the parent class](/assets/img/2022-04-10-retain-cycle.png)

This loop _can_ be broken. If `finishedPlaying` was reassigned to an empty closure after invocation, the cycle would be broken and the memory would be reclaimed. That requires a lot of awareness on our part to be very careful of when this can occur, and when to eliminate it.

## Rule 1: Strong `self` Isn't Always a Retain Cycle
While passing a strongly-retained `self` to a closure is a really good way to accidentally make a retain cycle, it's not a guarantee that there is one. In fact, the compiler is trying to help us use memory correctly. It makes a distinction between escaping and non-escaping closures.

### Escaping vs Non-escaping Closures
You may have written a method that takes a closure, only to be yelled at by the compiler:

![Assigning non-escaping parameter 'closure' to an @escaping closure](/assets/img/2022-04-10-missing-@escaping-annotation.png)

> Error: Assigning non-escaping parameter 'closure' to an @escaping closure.

This annotation is required by the compiler if the closure argument to your method has a lifetime beyond that of the method. In other words, does it escape the curly braces of your `function`?

If it doesn't, then when the method returns, we know nothing can retain that method. If nothing can own that closure, then it can't be part of a retain cycle, no matter what it strongly captures. In other words, **it's always safe to use a strong `self` in a non-`@escaping` closure.**

![A diagram showing a parent class retaining its child, that child retaining its closure, and the closer retaining nothing.](/assets/img/2022-04-10-non-escaping-closure.png)

### A Non-escaping Child Method
Let's see this in action with our code above. If we can guarantee that we're going to be done with the completion method before the method returns, we can remove the `@escaping` annotation. Let's write a non-escaping `play(completion:)` method:

```swift
extesion Child {
    func play(completion: () -> Void) {
        // 🏓
        completion()
    }
}
```

Using this method from `Parent`, we can see it in action:

```swift
extension Parent {
    func playChild() {
        child.play {
            self.didChildPlay = true
        }
    }
}
```

Better yet, we don't need to specify the `self` keyword. The closure _still captures self_, but the compiler knows it won't create a retain cycle, so it doesn't require you to specify it explicitly

```diff
 extension Parent {
     func playChild() {
         child.play {
-            self.didChildPlay = true
+            didChildPlay = true
         }
     }
 }
```

This is one of the ways the compiler helps us. If we can get by without writing `self.`, then we can be certain this closure won't be retained as part of a retain cycle.

## Rule 2: Weak `self` Will Never Be A Retain Cycle
Maybe you prefer to see `self.`, even when it's not required, to make capture semantics explicit. Now you need decide if `self` should be captured weakly or not. The real question from the first Golden Rule is this: are you _sure_ the closure in that method isn't `@escaping`?

- Did you check the documentation with each closure you created?
- Are you sure the documentation matches the implementation?
- Are you sure the implementation didn't change when you updated dependencies?

If any of those questions planted the seed of doubt, you'll see why the technique of using `[weak self]` everywhere you use a closure is so popular. Let's use `weak self` with our `playLater(completion:)` method:

```swift
class Parent {
    // ...
    func playChildLater() {
        child.playLater { [weak self] in
            self?.didChildPlay = true
        }
    }
}
```

It doesn't matter how that closure is passed around, retained, or if it's `@escaping` or not. That closure doesn't capture a strong reference to the Parent class, so we're sure it won't create a retain cycle.

![A diagram showing a parent class retaining its child, that child retaining noting, and a closure retaining the parent class](/assets/img/2022-04-10-weak-self.png)

## Rule 3: Upgrade `self` to Avoid Weird Behavior
If we follow the 2nd rule, then we're going to have to work with a lot of `weak self`s everywhere. This can become cumbersome. The standard advice is to use a `guard let` statement to upgrade `self` to a strong reference at the top of the closure, like this:

```swift
class Parent {
    // ...
    func playChildLater() {
        child.playLater { [weak self] in
            guard let self = self else { return }
            self.didChildPlay = true
        }
    }
}
```

But, why? Why not...
- Use `strongSelf` so I can keep a weak reference around?
- Just use the weak self repeatedly in my code?

### Using `strongSelf` Instead of `self`
Consider the following bit of code:

```swift
class Parent {
    // ...
    let firstChild = Child()
    let secondChild = Child()
    func playWithChildren(completion: @escaping (Int) -> Void) {
        firstChild.playLater { [weak self] in
            guard let strongSelf = self else {
                return
            }
            strongSelf.gamesPlayed += 1
            strongSelf.secondChild.playLater {
                if let strongSelf = self {
                    print("Played \(self?.gamesPlayed ?? -1) with first child.")
                }
                strongSelf.gamesPlayed += 1
                completion(strongSelf.gamesPlayed)
            }
        }
    }
}
```

Here, we name our upgraded `self` "`strongSelf`" so we can still pass a weak reference to a later method. This code works, but increases the complexity of the code you have to write. Anytime there's increased complexity, there's a greater chance for sneaky bugs to appear.

For instance, did you notice:
- `strongSelf` isn't syntax-highlighted like `self` is, making it harder to see
- `self?.gamesPlayed ?? -1` is used where `strongSelf.gamesPlayed` could have been used
- `strongSelf` is accidentally captured in the inner closure, **causing a retain cycle in a closure that used `weak self`**

😨

> You might see this and think, "Yeah, but I wouldn't write that code." Maybe not! Are you sure your whole team understand this nuance? I've had to fix bugs like this with `strongSelf` on teams of strong coders. Bugs like this happen. Why not let the tools do their best to make finding them easy?

### I'll Just Use `self?` Everywhere Then
Let's presume I've scared you off of `strongSelf`. Consider the following code:

```swift
class Parent {
    // ...
    let points = 1
    let firstChild = Child()
    func awardPoints(completion: @escaping (Int) -> Void) {
        firstChild.playLater { [weak self] in
            var totalPoints = 0
            totalPoints += self?.points ?? 0 // 1️⃣
            totalPoints += self?.points ?? 0 // 2️⃣
            completion(totalPoints)
        }
    }
}
```

This works, and is completely safe, but can lead to some odd behavior that you might not expect.

While `self` is weak, it's not increasing the retain count on `self`. That means that, at any point, the object that _is_ retaining `self` could release it. Since this is a multithreaded environment, that might happen in the middle of your closure. In other words, any reference to `self?` could be the first one to return `nil` for the rest of your method.

It's possible that the completion could be
- Called with `0` points
- Called with `2` points
- Called with `1` point

Wait... what? A resulting point total of `1` looks like a bug. It can happen when `self` becomes `nil` after line 1️⃣ runs, but before line 2️⃣ runs. In fact, each access to `self?` creates a branch in your code, before which `self` exists, and after which it's nil.

That's a lot more complexity than we usually mean to create. Usually, we just want to avoid a retain cycle, and have the closure execute all the way through. Good news: **you can enforce the all-or-nothing-ness of your closure by upgrading `self` to a strong reference at the top of your closure.**

```swift
class Parent {
    // ...
    let points = 1
    let firstChild = Child()
    func awardPoints(completion: @escaping (Int) -> Void) {
        firstChild.playLater { [weak self] in
            guard let self = self else {
                completion(0)
                return
            }
            var totalPoints = 0
            totalPoints += self.points
            totalPoints += self.points
            completion(totalPoints)
        }
    }
}
```


Now there's only one branch where `self` might be nil, and it's out of the way early. Either `self` has already become `nil` before this closure runs, or it's guaranteed to exist for its duration. The completion will be called with either `2` or `0`, but it can never be called with `1`.

## Let's Recap
As I said, this stuff isn't easy to reason about. If you want to do as little reasoning as possible, follow the three rules:

1. Only use a strong `self` for non-`@escaping` closures (ideally, omit it & trust the compiler)
1. Use `weak self` if you're not sure
1. Upgrade `self` to a strongly-retained `self` at the top of your closure.

These rules might be repetitive, but they lead to the safest code that's easiest to reason about. And they're pretty easy to ~~retain~~ remember.
