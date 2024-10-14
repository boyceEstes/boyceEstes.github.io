# Singletons For Swift 6 Concurrency

## Singletons

A **singleton** is a class that ensures only one instance of itself is ever created, providing a global access point throughout the app. It’s achieved by making the initializer private and exposing the instance through a static property.

This pattern is often used to manage shared resources such as network managers, logging systems, or caching services. While useful, singletons can introduce complications related to **reusability**, **testability**, and—especially with mutable state—**concurrency safety**.

## Mutable State
A key challenge with singletons arises when they hold **mutable state**, i.e., properties that can be modified. When multiple threads or tasks try to access or modify this state simultaneously, it can lead to **data races**.

### The Problem
Historically, there is an a sticking point, especially for beginners, where you might accidentally introduce a data race with your singleton. This means that you could have two threads in your app trying to access/modify a property in your singleton at the same exact time. 

Take this example: a NetworkRequestManager singleton that tracks the number of active network requests. If multiple network requests start simultaneously, they may both try to modify the activeRequests property at the same time.

```
final class NetworkRequestManager {
    static let shared = NetworkRequestManager()

    private init() {}

	// Two `startRequests()` calls in different threads at the same time updates to "1"
	// instead of "2"
    var activeRequests: Int = 0

    func startRequest() {
        activeRequests += 1
        // Code to start the network request...
    }

    func endRequest() {
        activeRequests -= 1
        // Code to end the network request...
    }
}
```

If two threads call startRequest() simultaneously, the activeRequests value might be incorrectly updated due to a **race condition**. This results in an inconsistent state where activeRequests might show 1 when there are actually two ongoing requests.

### Modern Swift Concurrency
With Swift 6, concurrency rules have become stricter to help prevent data races. When attempting to create a basic singleton like the one below, Swift will raise a warning if it detects potential concurrency issues:

Let’s imagine we’re making a little text-based video game and we want to have a Singleton that multiple types can access to “attack” 

```
final class AttackManager {
    
    static let shared = AttackManager()
    
    private init() {}
}

// Error: Static property 'shared' is not concurrency-safe because non-'Sendable' type 
```


So let's break this error down.

- Swift has some safe-guards for anything that is not **concurrency-safe** now. Which is a good thing! It helps prevent data race problems (two different threads trying to modify the same data at the same time leading to unpredictable results) from sneaking up on you.

- **Sendable** is the magic word that denotes if something is safe to cross concurrency boundaries. Crossing a concurrency boundary refers to passing or accessing resources between different concurrent tasks (think `Task {}`, `async` functions, or `actors`)

By creating a static instance of `AttackManager`, we are setting off alarm bells because many different contexts could access this one instance, possibly at the same time, but the instance is not marked as safe for concurrency. 

### Solutions to Ensure Concurrency Safety in Singletons
There are a few ways to solve this:
1. Simply make your Singleton an actor. This makes sure that Swift knows that it safe because the `actor` logic guarantees that it manages multiple calls safely
2. Mark the Singleton with `@unchecked Sendable` - This will says, “No don’t worry about it. The whole type (the class in this case) is thread-safe. I promise.” And now Swift will think its safe like any actor.
3. Mark as `static nonisolated(unsafe) let shared` - similar to 2, this says “Yeah, it could lead to concurrency issues but I’ve got it taken care of. Ignore this.” This is useful for singletons. You are aware that concurrent access could occur lead to data races and you take responsibility for it. It does not enforce safety checks for shared mutable state aka your type singleton object
4. Mark as static shared property as `@MainActor`  - This ensures that the instance is only accessed from the main queue which necessitates that it happens one at a time and there can never be two threads accessing or modifying at the same time. Making it *safe*


### 1. Use an Actor
No error, thread-safe, Sendable-conforming object.

The most straightforward way to make a singleton thread-safe is to use an actor. Actors in Swift guarantee that only one task can access their mutable state at any given time.

```
actor AttackManager {
    
    static let shared = AttackManager()
    
    private init() {}
}
```

With actor, Swift handles the concurrency automatically, ensuring that mutable states are protected against simultaneous access.

### 2. @unchecked Sendable
No error, “I promise this type is thread-safe, you don’t have to worry about it” 

```
final class AttackManager: @unchecked Sendable {

    static let shared = AttackManager()

    private init() {}
}
```

This approach should be used with caution because there is not any guardrail to give you a warning if there is a concurrency issue. You would probably want to use traditional mechanisms to implement concurrency like Global Dispatch Queue or some kind of semaphore/lock. Or you could simply ensure that there is not a mutable state.

### 3. Nonisolated (unsafe)
No error, “I promise this property is safe if its accessed concurrently”. 

```
final class AttackManager {

    nonisolated(unsafe) static let shared = AttackManager()

    private init() {}
}
```

Swift knows that there is potential for this `shared` instance to be referenced at the same time from different contexts, which could cause a data race if not handled correctly. Again, you would want to ensure that mutable states are modified appropriately using traditional mechanisms.

### 4. Main Actor
Just run any access of this instance serially (one at a time) on the Main Queue. This is okay if your Singleton should be doing UI-work or processing using very little computational time. Otherwise probably the last resort.

```
final class AttackManager {

    @MainActor static let shared = AttackManager()

    private init() {}
}
```

## Conclusion
Singletons are a useful design pattern, but they introduce complexity when dealing with **concurrency** and **mutable state**. Swift 6 provides robust tools to ensure safety, but as a developer, it’s essential to choose the right approach—whether it’s using an actor, @unchecked Sendable, nonisolated, or @MainActor—to avoid subtle concurrency bugs in your app.
