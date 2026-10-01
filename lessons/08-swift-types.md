---
id: swift-types
title: Structs, Classes & Enums
part: 2
minutes: 40
summary: Model your app's data with Swift's three kinds of types. Understand value vs. reference semantics, properties and methods, and enums with associated values, which power every SwiftUI screen.
---

:::goals
- Structs: properties, methods, initialisers, `mutating`
- Classes: reference semantics, inheritance, `final`
- **Value vs. reference types**, the key difference
- Computed properties and property observers
- Enums with raw values and **associated values**
- Static members, and choosing between struct, class and enum
:::

## Structs

A **struct** groups related data and behaviour into a new type. Most of your models, and every SwiftUI view, will be structs.

```swift runnable
struct Recipe {
    let name: String            // stored properties
    var minutes: Int
    var isFavorite = false      // default value

    func summary() -> String {                   // a method
        "\(name) · \(minutes) min" + (isFavorite ? " ♥" : "")
    }

    mutating func toggleFavorite() {             // mutating: it changes self
        isFavorite.toggle()
    }
}

var pancakes = Recipe(name: "Pancakes", minutes: 20)   // free "memberwise" initialiser
pancakes.minutes = 25
pancakes.toggleFavorite()
print(pancakes.summary())
```

- Swift generates a **memberwise initialiser** automatically: `Recipe(name:minutes:isFavorite:)`, where parameters with defaults are optional.
- Methods that modify the struct must be marked **`mutating`**.
- A struct stored in a `let` constant can't change at all, even its `var` properties.

## Classes

A **class** is similar, but with important differences: classes are **reference types**, support **inheritance**, and need an initialiser if their properties have no defaults.

```swift runnable
class Animal {
    let name: String
    init(name: String) {                  // classes need an explicit init
        self.name = name
    }
    func sound() -> String { "..." }
    func describe() -> String { "\(name) says \(sound())" }
}

final class Cat: Animal {                 // final = can't be subclassed further
    override func sound() -> String { "Meow" }
}

class Cow: Animal {
    override func sound() -> String { "Moo" }
}

let animals: [Animal] = [Cat(name: "Tom"), Cow(name: "Daisy")]
for animal in animals {
    print(animal.describe())
}
```

## ⭐ Value types vs. reference types

This is one of the most important ideas in Swift.

- **Structs, enums, arrays, dictionaries, strings** are **value types**: assigning or passing them **copies** the value.
- **Classes** are **reference types**: assigning or passing them shares **the same object**.

```swift runnable
struct PointStruct { var x = 0 }
class PointClass { var x = 0 }

var a = PointStruct()
var b = a            // copy
b.x = 99
print("struct:", a.x, b.x)   // 0 99: a is unaffected

let c = PointClass()
let d = c            // same object
d.x = 99
print("class:", c.x, d.x)    // 99 99: both names point to one object
```

<div class="diagram"><div class="flow">
<div class="hl">Value type<small>each variable owns its own copy</small></div>
<div>Reference type<small>variables share one object in memory</small></div>
</div></div>

:::tip Default to structs
Apple's guidance: **use structs by default**. They're simpler, safer (no surprising shared changes) and fast. Reach for a class when you need shared, identity-based state that several parts of the app observe and change. A typical example is an `@Observable` view model or data store (Lesson 21).
:::

## Computed properties & property observers

```swift runnable
struct Rectangle {
    var width: Double
    var height: Double {
        didSet { print("Height changed from \(oldValue) to \(height)") }   // observer
    }

    var area: Double { width * height }            // computed: calculated every time

    var isSquare: Bool { width == height }
}

var r = Rectangle(width: 3, height: 4)
print(r.area)
r.height = 3
print(r.isSquare)
```

- A **computed property** has no storage; it calculates its value each time it's read.
- `willSet` / `didSet` run code before or after a stored property changes.

## Custom initialisers

```swift runnable
struct Temperature {
    var celsius: Double

    init(celsius: Double) {
        self.celsius = celsius
    }

    init(fahrenheit: Double) {                    // another way to create one
        celsius = (fahrenheit - 32) * 5 / 9
    }

    init?(text: String) {                         // failable init: may return nil
        guard let value = Double(text) else { return nil }
        celsius = value
    }
}

print(Temperature(fahrenheit: 212).celsius)
print(Temperature(text: "21.5")?.celsius ?? .nan)
print(Temperature(text: "hot")?.celsius ?? .nan)
```

## Enums

An **enum** defines a type with a fixed set of cases:

```swift runnable
enum Difficulty: String, CaseIterable {      // raw values + list of all cases
    case easy, medium, hard

    var stars: String {                      // enums can have computed properties
        switch self {
        case .easy: "★"
        case .medium: "★★"
        case .hard: "★★★"
        }
    }
}

let d = Difficulty.medium
print(d.rawValue, d.stars)                    // "medium ★★"
print(Difficulty(rawValue: "hard") ?? .easy)  // create from a raw value (optional)
print(Difficulty.allCases.map(\.rawValue))    // from CaseIterable

var level: Difficulty = .easy                 // type known, so .easy is enough
level = .hard
print(level)
```

The leading-dot syntax (`.easy`) works whenever Swift already knows the type. That's why SwiftUI code is full of things like `.padding(.horizontal)` and `.font(.title)`.

### Associated values ⭐

Each case can carry its **own data**. This is perfect for modelling states, such as what a screen is showing:

```swift runnable
enum LoadState {
    case loading
    case loaded([String])
    case failed(message: String)
}

func render(_ state: LoadState) -> String {
    switch state {                                // exhaustive: every case handled
    case .loading:
        "⏳ Loading…"
    case .loaded(let items) where items.isEmpty:
        "📭 Nothing here yet"
    case .loaded(let items):
        "✅ \(items.count) items: \(items.joined(separator: ", "))"
    case .failed(let message):
        "❌ \(message)"
    }
}

print(render(.loading))
print(render(.loaded(["Pasta", "Curry"])))
print(render(.loaded([])))
print(render(.failed(message: "No internet")))
```

Because `switch` must be exhaustive, if you later add a case (say `.offline`), every `switch` that doesn't handle it becomes a compile error. You can't forget to update the UI.

## Static members

`static` properties and methods belong to the **type**, not to instances:

```swift runnable
struct AppConfig {
    static let apiURL = "https://dummyjson.com"
    static let maxRetries = 3
}

struct Recipe {
    var name: String
    static let sample = Recipe(name: "Sample Pancakes")    // handy for previews!
}

print(AppConfig.apiURL)
print(Recipe.sample.name)
```

## Struct, class or enum?

| Use a… | When |
|---|---|
| **struct** | Data and most models: a recipe, a user, settings, **SwiftUI views** |
| **enum** | A fixed set of options or states: difficulty, a loading state, a tab |
| **class** | Shared mutable state with identity: `@Observable` view models and stores, or when an Apple API requires it (UIKit) |

## Recap

:::recap
- Structs are value types (copied); classes are reference types (shared). Default to structs.
- `mutating` methods change a struct; `final` stops subclassing; `override` replaces superclass behaviour.
- Computed properties calculate values; `didSet` reacts to changes; `init?` can fail.
- Enums have cases, raw values and associated values, and `switch` over them is exhaustive.
:::

## Practice

:::exercise
1. Create `struct Book` with `title`, `author`, `pages` and `isRead` (default `false`), plus a `mutating func markRead()`. Make an array of four books, mark one as read, and print the total pages read.
2. Create `enum PaymentResult` with cases `success(amount: Double)`, `declined(reason: String)` and `pending`, and a function that returns a friendly message for each.
3. Demonstrate value semantics: copy a `Book` into a new variable, change the copy's title, and print both titles.
:::

:::solution Show solution
```swift runnable
struct Book {
    let title: String
    let author: String
    let pages: Int
    var isRead = false

    mutating func markRead() { isRead = true }
}

enum PaymentResult {
    case success(amount: Double)
    case declined(reason: String)
    case pending
}

func message(for result: PaymentResult) -> String {
    switch result {
    case .success(let amount): "Paid $\(amount). Thank you!"
    case .declined(let reason): "Payment declined: \(reason)"
    case .pending: "Payment is processing…"
    }
}

var books = [
    Book(title: "Dune", author: "Herbert", pages: 412, isRead: true),
    Book(title: "The Swift Programming Language", author: "Apple", pages: 800),
    Book(title: "Clean Code", author: "Martin", pages: 464),
    Book(title: "The Hobbit", author: "Tolkien", pages: 310)
]
books[3].markRead()
print("Pages read:", books.filter(\.isRead).reduce(0) { $0 + $1.pages })

print(message(for: .success(amount: 9.99)))
print(message(for: .declined(reason: "Insufficient funds")))
print(message(for: .pending))

let original = books[0]
var copy = original
copy.isRead = false
print(original.isRead, copy.isRead)   // true false: independent copies
```
:::

## Check your understanding

```quiz
Q: After `var b = a` where `a` is a struct, you change `b.x`. What happens to `a.x`?
- [x] Nothing, because b is an independent copy
- [ ] It changes too
- [ ] It becomes nil
- [ ] Compile error
> Structs are value types: assignment copies them.

Q: Why must some struct methods be marked `mutating`?
- [ ] To make them faster
- [x] Because they modify the struct's own properties, which Swift requires you to mark explicitly
- [ ] So they can be overridden
- [ ] To make them asynchronous
> Structs are immutable by default inside methods; mutating opts in to changing self.

Q: Which is the best fit for "a screen that's loading, showing results, or showing an error message"?
- [ ] A class with three Booleans
- [x] An enum with associated values, e.g. `.loading`, `.loaded([Item])`, `.failed(String)`
- [ ] A dictionary
- [ ] A tuple
> Enums make impossible states impossible, and switch forces you to handle every case.

Q: What is Apple's general advice for choosing between struct and class?
- [ ] Always use classes
- [x] Default to structs; use classes when you need shared, identity-based state
- [ ] Use structs only for numbers
- [ ] They're interchangeable
> Value semantics are simpler and safer; classes are for shared mutable state.
```
