---
id: swift-concurrency
title: "Concurrency: async/await & Actors"
part: 2
minutes: 40
summary: iOS apps constantly wait for the network, the disk and the database. Swift Concurrency lets you write that code simply and safely, without freezing the UI.
---

:::goals
- Why the main thread must never be blocked
- `async` functions and `await`
- Running work in parallel with `async let` and task groups
- `Task`, cancellation and the `.task` modifier preview
- The main actor, actors and data-race safety in Swift 6
- `AsyncSequence` basics
:::

## The problem: the main thread

Every iOS app has a **main thread** that draws the UI and handles touches, 60–120 times per second. If it has to wait for something slow, like a network request or a big file, the app freezes and feels broken. iOS may even terminate an app that hangs for too long.

**Swift Concurrency** lets you write asynchronous code that reads top to bottom, while the system keeps the UI responsive:

```swift
func loadProfile() async throws -> Profile {
    let user = try await api.fetchUser()          // suspends while waiting, doesn't block
    let posts = try await api.fetchPosts(for: user.id)
    return Profile(user: user, posts: posts)
}
```

## `async` and `await`

- An **`async`** function can *suspend*: pause while it waits, freeing the thread for other work, and resume later.
- You call it with **`await`**, which marks each point where suspension can happen.

```swift runnable
func fetchUserName() async -> String {
    try? await Task.sleep(for: .seconds(1))   // pretend network delay (suspends, doesn't block)
    return "Ada"
}

print("Fetching…")
let name = await fetchUserName()              // top-level code can await in a script
print("Got \(name)")
```

Functions can be both: `async throws`, called with `try await`.

:::analogy
A blocked thread is a chef staring at the oven for 20 minutes. With `await`, the chef puts the dish in, goes off to chop vegetables for other orders, and comes back when the timer rings.
:::

## Parallel work: `async let`

`await` calls run one after another. To start several at once and wait for all of them, use **`async let`**:

```swift runnable
import Foundation

func fetchWeather() async -> String { try? await Task.sleep(for: .seconds(1)); return "☀️ 24°C" }
func fetchNews() async -> String { try? await Task.sleep(for: .seconds(1)); return "📰 Local team wins" }

let clock = ContinuousClock()

let sequential = await clock.measure {
    let w = await fetchWeather()
    let n = await fetchNews()
    print(w, "|", n)
}
print("Sequential:", sequential)        // ~2 seconds

let parallel = await clock.measure {
    async let w = fetchWeather()        // starts immediately
    async let n = fetchNews()           // starts immediately
    print(await w, "|", await n)        // wait for both
}
print("Parallel:", parallel)            // ~1 second
```

For a *variable* number of parallel jobs (e.g. download 20 images), use a **task group**:

```swift runnable
func price(of item: String) async -> Double {
    try? await Task.sleep(for: .milliseconds(300))
    return Double(item.count)
}

let total = await withTaskGroup(of: Double.self) { group in
    for item in ["apple", "bread", "cheese"] {
        group.addTask { await price(of: item) }
    }
    var sum = 0.0
    for await p in group { sum += p }
    return sum
}
print("Total:", total)
```

## `Task`: starting async work from sync code

You can only `await` inside an async context. To start async work from normal code, such as a button tap, create a **`Task`**:

```swift
Button("Refresh") {
    Task {
        await viewModel.refresh()
    }
}
```

In SwiftUI you'll more often use the **`.task` modifier**, which starts work when a view appears and **cancels it automatically** when the view disappears:

```swift
List(recipes) { … }
    .task {                          // runs when the view appears
        await viewModel.load()
    }
```

### Cancellation

Tasks can be cancelled; well-behaved async code checks and stops early. `Task.sleep` and `URLSession` throw `CancellationError` when cancelled.

```swift runnable
let task = Task {
    for i in 1...10 {
        guard !Task.isCancelled else { print("Stopped early"); return }
        print("Working \(i)…")
        try? await Task.sleep(for: .milliseconds(200))
    }
}
try? await Task.sleep(for: .milliseconds(700))
task.cancel()
_ = await task.value
print("Done")
```

## The main actor

UI code must run on the main thread. In Swift this is expressed with the **main actor**: mark a type or function **`@MainActor`** and Swift guarantees it runs on the main thread. The compiler enforces it.

```swift
@MainActor
@Observable
final class RecipesViewModel {
    var recipes: [Recipe] = []           // UI state: only touched on the main actor

    func load() async {
        do {
            recipes = try await service.fetchRecipes()   // the network call suspends; the UI stays responsive
        } catch {
            // show an error
        }
    }
}
```

SwiftUI views run on the main actor. And **new app projects in recent Xcode versions make the main actor the default** for your app's code (a build setting called *Default Actor Isolation*). That keeps beginner code simple and safe: everything runs on the main actor unless you opt out, and slow `await`ed work (networking, file I/O done by Apple's APIs) happens elsewhere without blocking it.

## Actors and data-race safety

A **data race** happens when two threads access the same mutable data at the same time, causing random, hard-to-find bugs. Swift 6 **checks for data races at compile time**.

An **`actor`** is like a class that protects its mutable state: only one task can use it at a time, and outside code must `await` to access it.

```swift runnable
actor ImageCache {
    private var storage: [String: String] = [:]

    func image(for url: String) -> String? { storage[url] }
    func insert(_ image: String, for url: String) { storage[url] = image }
    var count: Int { storage.count }
}

let cache = ImageCache()

await withTaskGroup(of: Void.self) { group in
    for i in 1...100 {                         // 100 concurrent writes, zero data races
        group.addTask { await cache.insert("img\(i)", for: "url\(i)") }
    }
}
print(await cache.count)                       // always 100
print(await cache.image(for: "url42") ?? "missing")
```

Values passed between concurrent code must be **`Sendable`**, meaning safe to share. Structs and enums made of `Sendable` values are automatically `Sendable`. When the compiler complains about sending something non-Sendable across actors, it's catching a real potential bug.

:::tip Don't panic about concurrency warnings
Swift 6's checks can feel strict at first. The usual fixes: keep UI state in `@MainActor` types, use structs for data passed around, and use an `actor` for shared mutable state that background work touches.
:::

## `AsyncSequence`: values over time

An `AsyncSequence` delivers values over time, and you loop over it with `for await`:

```swift runnable
func countdown(from n: Int) -> AsyncStream<Int> {
    AsyncStream { continuation in
        Task {
            for i in stride(from: n, through: 0, by: -1) {
                continuation.yield(i)
                try? await Task.sleep(for: .milliseconds(300))
            }
            continuation.finish()
        }
    }
}

for await value in countdown(from: 3) {
    print("T-minus \(value)")
}
print("🚀")
```

Apple APIs expose many of these: notifications, location updates, lines of a file, bytes of a download.

## Recap

:::recap
- Never block the main thread; `await` suspends without blocking.
- `async let` and task groups run work in parallel.
- `Task { }` starts async work from sync code; SwiftUI's `.task` ties work to a view's lifetime and cancels it automatically.
- `@MainActor` code runs on the main thread, which is where UI state lives.
- `actor` protects shared mutable state; Swift 6 catches data races at compile time.
:::

## Practice

:::exercise
1. Write `func fetchScore(for player: String) async -> Int` that sleeps for 500 ms and returns `player.count * 10`. Fetch scores for three players **in parallel** with `async let` and print the total. Measure it with `ContinuousClock`.
2. Create an `actor Counter` with `increment()` and `value`. Increment it 1,000 times from a task group and print the final value.
:::

:::solution Show solution
```swift runnable
func fetchScore(for player: String) async -> Int {
    try? await Task.sleep(for: .milliseconds(500))
    return player.count * 10
}

actor Counter {
    private(set) var value = 0
    func increment() { value += 1 }
}

let elapsed = await ContinuousClock().measure {
    async let a = fetchScore(for: "Ada")
    async let b = fetchScore(for: "Grace")
    async let c = fetchScore(for: "Linus")
    let total = await a + b + c
    print("Total score:", total)
}
print("Took", elapsed, "(about 0.5s, not 1.5s)")

let counter = Counter()
await withTaskGroup(of: Void.self) { group in
    for _ in 1...1000 {
        group.addTask { await counter.increment() }
    }
}
print("Counter:", await counter.value)
```
:::

## Check your understanding

```quiz
Q: What does `await` do?
- [ ] Blocks the thread until the result arrives
- [x] Marks a point where the function may suspend while waiting, freeing the thread for other work
- [ ] Runs code on a background thread
- [ ] Cancels a task
> Suspension is not blocking: other work continues on the thread.

Q: You need to call two independent async functions and use both results. How do you run them at the same time?
- [ ] Call them with await one after another
- [x] Use `async let` for each, then await both
- [ ] Use DispatchQueue.main.sync
- [ ] Wrap each in try!
> async let starts child tasks immediately; awaiting collects the results.

Q: What's the advantage of SwiftUI's `.task` modifier over `Task { }` in `onAppear`?
- [ ] It runs faster
- [x] The work is automatically cancelled when the view disappears
- [ ] It runs on a background thread
- [ ] It retries on failure
> .task ties the work's lifetime to the view's lifetime.

Q: What does an `actor` guarantee?
- [ ] Its code runs on the main thread
- [x] Only one task at a time can access its mutable state, preventing data races
- [ ] It never suspends
- [ ] It's always faster than a class
> Actors serialise access to their state; outside callers must await.

Q: Where should UI state like a view model's `recipes` array live?
- [ ] In an actor on a background thread
- [x] In a `@MainActor` type, since UI updates must happen on the main thread
- [ ] In a global variable
- [ ] Anywhere, SwiftUI doesn't care
> SwiftUI reads state on the main actor; keeping UI state there avoids data races.
```
