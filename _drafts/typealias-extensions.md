---
layout: post
title:  Typealias Extensions for Complex Types
date:   2022-05-11 08:00:00 -0700
category: draft
tags: swift ios
---

Swift is an incredibly extensible language. You can add capabilities to existing types with `extension`s. You can create your own nicknames for types with `typealias`. But did you know you can use these language features _at the same time_ to make extending complex types easier?

Let's take a look!

## Looking Up IDs
Let's say we have an app that keeps a dictionary that maps from names to IDs:

```swift
var idLookup = [String: Int]()

idLookup["Ava"] = 1
idLookup["Janine"] = 2
idLookup["Melissa"] = 3
```

We'd like to make our system optimistic -- when we perform a lookup, if the name doesn't exist, we'd like it to be created with a new, unique ID. We already have a function that does this by finding the highest ID and incrementing it by 1 if the user doesn't exist.

```swift
func findOrCreate(name: String) -> Int {
    if let id = idLookup[name] {
        return id
    }
    let newId = Array(idLookup.values).reduce(Int.min, max) + 1
    idLookup[name] = newId
    return newId
}

findOrCreate(name: "Janine", in: idLookup) // 2
findOrCreate(name: "Jacob", in: idLookup) // 4
```

> Side note: Generating and storing IDs in an app is probably a Very Bad Idea to do in a production app, especially if they need to communicate with a server. Let's ignore that engineering malfeasance for now, though.

## Making it a Method
It would be a lot nicer if we could get this behavior by calling a method on the dictionary, like this:

```swift
idLookup.findOrCreate(name: "Ava")
```

Using an extension, we can add this method to the dictionary type:

```swift
extension [String: Int] {
    mutating func findOrCreate(name: String) -> Int {
        if let id = self[name] {
            return id
        }
        let newId = Array(values).reduce(Int.min, Swift.max) + 1
        self[name] = newId
        return newId
    }
}
```

... well, kind of. Declaring the extension this way (or as an extension of the `Dictionary<String, Int>` type) generates an error:

```
Constrained extension must be declared on the unspecialized generic type 'Dictionary' with constraints specified by a 'where' clause
```

To have this extension stick, we need to follow the error's advice: extend the `Dictionary` type, and then add constraints for the type that the `Key` and `Value` generic types could be. This limits the extension to only apply to that specific dictionary type.

```swift
extension Dictionary where Key == String, Value == Int {
    mutating func findOrCreate(name: String) -> Int {
       // ...
    }
}
```

That's quite of mouthful of syntax. It's also arguably harder to read than the `extension [String: Int]` that we first tried to write. Let's use a `typealias` to make it better.

## `typealias` to the Rescue
Type aliases are a great way to nickname common types in your code for clarity. If we declare a type alias, then we can extend the alias instead and get all the sweet syntactic sugar we like.

```swift
typealias IDLookup = [String: Int]
extension IDLookup {
    mutating func findOrCreate(name: String) -> Int {
        // ...
    }
}
```

### Alias Scoping
While this works great, it has a bit of a side effect. The alias we've created has a default `internal` level of access - it can be seen elsewhere in our framework or app that uses this code.

In some cases, that might be what we want. It would especially fit our use case if we had already defined stronger, more specific types for the keys & values in our dictionary. This alias and its types, are specific enough where a module-wide `typealias` might make sense.

```swift
typealias AccountIDLookup = [Username: AccountID]
```

Unfortunately, that's not the example we're currently working with. A more generic `[String: Int]` type might be used for things other than this "name to ID" data structure. Exposing our `IDLookup` type alias from before isn't 

===

To do that, we can do something that looks odd. We can declare the `typealias` private, but make its extension `internal` or `public`. Anything added in an extension this way _will have the scope of the `extension`, not the `typealias`_. It will outlive the `typealias` declaration.

```swift
private typealias IDLookup = [String: Int]
public extension IDLookup {
    mutating func findOrCreate(name: String) -> Int {
        // ...
    }
}
```