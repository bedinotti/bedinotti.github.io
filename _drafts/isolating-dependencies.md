---
layout: post
title:  Isolating Dependencies
date:   2021-08-21 00:00:00 -0700
category: draft
tags: draft
---

There are 4 steps to fully isolating your dependencies and making your existing code testable

1. Move singletons & shared instances into a private property
2. Set the property with an init argument, using a default for convenience
3. (Optional) Abstract the dependency behind a protocol. 
4. Create your own mock object (or mock subclasses) to inject during testing

Let's take a look at how to improve code like this in your app:

```swift
let task = URLSession.shared
    .dataTaskPublisher(for: URL(string: "http://google.com")!)
    .sink { completion in
        switch completion {
        case .finished:
            UserDefaults.standard.setValue(true, forKey: hasDownloadedResponseKey)
        case .failure(_):
            UserDefaults.standard.setValue(true, forKey: hasDownloadedResponseKey)
        }
    } receiveValue: { (data: Data, response: URLResponse) in
        UserDefaults().setValue(data, forKey: responseKey)
    }
```

## Make it a private property
Move these shared instances to a private property. It reduces duplication in your code and can even make it clear when you've accidentally mis-used a dependency, like our `recieveValue` completion not using the standard `UserDefaults`.

```swift
class Downloader {
    private let urlSession = URLSession.shared
    private let defaults = UserDefaults.standard
    
    func download() {
        let task = urlSession
            .dataTaskPublisher(for: URL(string: "http://google.com")!)
            .sink { [defaults] completion in
                switch completion {
                case .finished:
                    defaults.setValue(true, forKey: hasDownloadedResponseKey)
                case .failure(_):
                    defaults.setValue(true, forKey: hasDownloadedResponseKey)
                }
            } receiveValue: { [defaults] (data: Data, response: URLResponse) in
                defaults.setValue(data, forKey: responseKey)
            }
        // ...
    }
}
```

## Use an Initializer Argument
Now that our dependencies are clear, we can set them as arguments in an initializer that would let us inject alternative dependencies. This can be _incredibly_ useful for testing. However, since the most common usage is still to use these shared instances, we can set these as the default for those arguments

```swift
class Downloader {
    private let urlSession: URLSession
    private let defaults: UserDefaults

    init(urlSession: URLSession = URLSession.shared, 
         defaults: UserDefaults = UserDefaults.standard) {
        self.urlSession = urlSession
        self.defaults = defaults
    }
    // ... 
}
```

## Make it a Protocol
This `Downloader` type still depends on the concrete type of the shared instance. It can be quite tricky to correctly implement a custom subclass for testing. However, if you're only using a subset of that type's interface, then you can accept an initializer argument that conforms to a custom protocol that matches only what you need

```swift
protocol DataTaskPublishable {
    func dataTaskPublisher(for: URL) -> URLSession.DataTaskPublisher
}

extension URLSession: DataTaskPublishable {}

class Downloader {
    private let urlSession: DataTaskPublishable
    private let defaults: UserDefaults

    init(urlSession: DataTaskPublishable = URLSession.shared,
         defaults: UserDefaults = UserDefaults.standard) {
        self.urlSession = urlSession
        self.defaults = defaults
    }
}
```

Notice:
1. The new `DataTaskPublishable` only contains the methods we need
1. `URLSession` must conform to `DataTaskPublishable`. Since this method already exists on `URLSession`, conformance is met automatically.
1. The `urlSession` initializer argument and property now take a `DataTaskPublishable`, but _still_ defaults to `URLSession.shared`. 

It's also worth pointing out that you don't need to do this for all dependencies. You can specify a custom `UserDefaults` in a unit test quite easily. Mocking a `URLSession` is a bit harder. This makes it easier.

## Create Your Own Mock Objects
Now, the payoff. 
