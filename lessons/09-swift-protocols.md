---
id: swift-protocols
title: Protocols, Extensions & Generics
part: 2
minutes: 35
summary: The features at the heart of Swift and SwiftUI. Define contracts with protocols, add behaviour with extensions, write reusable code with generics, and understand `some` and `any`.
---

:::goals
- Defining and adopting protocols
- Protocol extensions and default implementations
- The standard protocols: `Equatable`, `Hashable`, `Comparable`, `Identifiable`, `Codable`
- Extensions on any type
- Generics, `some` and `any`
:::

## Protocols

A **protocol** is a contract: a list of properties and methods a type promises to provide. It says *what* something can do, not *how*.

```swift runnable
protocol Describable {
    var title: String { get }          // a read-only property requirement
    func describe() -> String          // a method requirement
}

struct Recipe: Describable {           // "Recipe conforms to Describable"
    let title: String
    let minutes: Int
    func describe() -> String { "\(title), ready in \(minutes) min" }
}

struct Article: Describable {
    let title: String
    let author: String
    func describe() -> String { "\(title) by \(author)" }
}

let things: [any Describable] = [
    Recipe(title: "Ramen", minutes: 30),
    Article(title: "Why Swift?", author: "Ada")
]
for thing in things {
    print(thing.describe())
}
```

Any type (struct, class or enum) can conform to any number of protocols. You've already used one: every SwiftUI view is a struct conforming to the `View` protocol, whose single requirement is `var body: some View`.

:::note Why protocols matter so much
Architecture relies on them. A view model can depend on a protocol like `RecipeService`: the real app passes an implementation that calls the network, while previews and tests pass a fake with sample data. You'll do exactly this in Parts 4, 5 and 7.
:::

## Protocol extensions: default implementations

You can **extend** a protocol to give every conforming type behaviour for free:

```swift runnable
protocol Greeter {
    var name: String { get }
}

extension Greeter {
    func greet() -> String { "Hello, I'm \(name)!" }   // every Greeter gets this
}

struct Person: Greeter { let name: String }
struct Robot: Greeter {
    let name: String
    func greet() -> String { "BEEP. UNIT \(name.uppercased()) ONLINE." }   // customise
}

print(Person(name: "Ada").greet())
print(Robot(name: "r2").greet())
```

This is how SwiftUI gives every view hundreds of modifiers like `.padding()` and `.font()`: they're defined in an extension on `View`.

## Standard protocols you'll use constantly

```swift runnable
struct Ingredient: Equatable, Hashable, Comparable, Identifiable, Codable {
    let id: Int
    let name: String

    static func < (lhs: Ingredient, rhs: Ingredient) -> Bool {   // Comparable
        lhs.name < rhs.name
    }
}

let flour = Ingredient(id: 1, name: "Flour")
let eggs = Ingredient(id: 2, name: "Eggs")

print(flour == Ingredient(id: 1, name: "Flour"))      // Equatable: == generated for you
print(Set([flour, eggs, flour]).count)                // Hashable: usable in sets and as dict keys
print([flour, eggs].sorted().map(\.name))             // Comparable: sorted() works
print(flour.id)                                       // Identifiable: has a stable id
```

| Protocol | Gives you | Used by |
|---|---|---|
| `Equatable` | `==` (usually generated automatically) | comparisons, `.onChange`, animations |
| `Hashable` | use in `Set` and as dictionary keys | navigation values, `ForEach` |
| `Comparable` | `<`, so `sorted()` and `max()` work | sorting |
| `Identifiable` | a stable `id` property | `List` and `ForEach` in SwiftUI |
| `Codable` | conversion to and from JSON | networking (Lesson 23) |
| `CustomStringConvertible` | a custom `description` for printing | debugging |

For structs and enums, Swift **synthesises** `Equatable`, `Hashable` and `Codable` automatically when all their properties conform. You just write the protocol name.

## Extensions

An **extension** adds functionality to any type, including ones you don't own, like `String`, `Int` or `Date`:

```swift runnable
extension Int {
    var minutesText: String {
        let h = self / 60, m = self % 60
        return h > 0 ? "\(h)h \(m)m" : "\(m) min"
    }
}

extension String {
    var isValidEmail: Bool {
        contains("@") && (split(separator: "@").last?.contains(".") ?? false)
    }
}

extension Array where Element == Int {              // only for arrays of Ints
    var average: Double { isEmpty ? 0 : Double(reduce(0, +)) / Double(count) }
}

print(135.minutesText)
print(45.minutesText)
print("ada@example.com".isValidEmail)
print("nope".isValidEmail)
print([4, 8, 6].average)
```

Extensions are also great for **organising** code: many developers put each protocol conformance in its own extension (`extension Recipe: Codable { … }`).

## Generics

**Generics** let you write code that works with any type while staying type-safe. You've been using them all along: `Array<Element>`, `Dictionary<Key, Value>`, `Optional<Wrapped>`.

```swift runnable
// A generic function: works for any T
func firstOrDefault<T>(_ items: [T], default fallback: T) -> T {
    items.first ?? fallback
}

// With a constraint: T must be Comparable
func largest<T: Comparable>(_ items: [T]) -> T? {
    items.max()
}

// A generic type
struct Box<Content> {
    let content: Content
    func describe() -> String { "Box containing \(content)" }
}

print(firstOrDefault([3, 4], default: 0))
print(firstOrDefault([String](), default: "nothing"))
print(largest(["pear", "apple", "zucchini"]) ?? "")
print(Box(content: 42).describe())
print(Box(content: ["a", "b"]).describe())
```

## `some` and `any`

You'll see both keywords in SwiftUI and modern Swift:

- **`some Protocol`** (an *opaque type*) means "one specific type that conforms, known to the compiler but not written out." `var body: some View` returns one concrete view type, which could be a `VStack` of many things, without you spelling out its huge generic type name. It's fast because the compiler knows the exact type.
- **`any Protocol`** (an *existential*) means "a box that can hold *any* conforming type, and which one can vary at runtime." Use it for mixed collections like `[any Describable]`, or for stored dependencies like `let service: any RecipeService`.

```swift runnable
protocol Shape { func area() -> Double }
struct Circle: Shape { let r: Double; func area() -> Double { 3.14159 * r * r } }
struct Square: Shape { let side: Double; func area() -> Double { side * side } }

func makeUnitSquare() -> some Shape { Square(side: 1) }     // always the same concrete type

let shapes: [any Shape] = [Circle(r: 1), Square(side: 2)]  // mixed types
print(makeUnitSquare().area())
print(shapes.map { $0.area() }.reduce(0, +))
```

:::tip Rule of thumb
Returning a view or a single concrete type → `some`. Storing different types together, or a dependency that could be real or fake → `any`.
:::

## Recap

:::recap
- A **protocol** is a contract; types conform by implementing its requirements.
- **Protocol extensions** provide default behaviour to every conforming type.
- `Equatable`, `Hashable`, `Identifiable` and `Codable` are often synthesised for free.
- **Extensions** add computed properties and methods to existing types.
- **Generics** (`<T>`) make reusable, type-safe code; `some` hides a single concrete type, `any` holds any conforming type.
:::

## Practice

:::exercise
1. Define `protocol Shape { var name: String { get }; func area() -> Double }` and add a protocol extension method `func summary() -> String` returning e.g. `"Circle: 3.14"`. Create `Circle` and `Rectangle` and print the summaries of an `[any Shape]`.
2. Add an extension to `Double` with a computed property `rounded2: String` that formats to two decimal places. (Hint: build it with `(self * 100).rounded() / 100`.)
3. Write a generic function `func countMatches<T: Equatable>(_ items: [T], of target: T) -> Int`.
:::

:::solution Show solution
```swift runnable
protocol Shape {
    var name: String { get }
    func area() -> Double
}

extension Shape {
    func summary() -> String { "\(name): \(area().rounded2)" }
}

struct Circle: Shape {
    let radius: Double
    var name: String { "Circle" }
    func area() -> Double { Double.pi * radius * radius }
}

struct Rectangle: Shape {
    let width, height: Double
    var name: String { "Rectangle" }
    func area() -> Double { width * height }
}

extension Double {
    var rounded2: String { String((self * 100).rounded() / 100) }
}

func countMatches<T: Equatable>(_ items: [T], of target: T) -> Int {
    items.filter { $0 == target }.count
}

let shapes: [any Shape] = [Circle(radius: 1), Rectangle(width: 2, height: 3.5)]
shapes.forEach { print($0.summary()) }
print(countMatches(["a", "b", "a"], of: "a"))
print(countMatches([1, 2, 3, 2, 2], of: 2))
```
:::

## Check your understanding

```quiz
Q: What is a protocol?
- [ ] A kind of class
- [x] A contract of properties and methods that conforming types must provide
- [ ] A network standard
- [ ] A SwiftUI view
> Types adopt protocols and implement their requirements.

Q: What does `var body: some View` mean?
- [ ] body can return a different view type each time it runs
- [x] body returns one specific concrete View type that the compiler knows but you don't have to write out
- [ ] body returns an optional view
- [ ] body is any object at all
> `some` is an opaque type: one fixed type hidden behind the protocol.

Q: Which protocol does SwiftUI's `ForEach` want your data to conform to so it can track items?
- [ ] Codable
- [ ] Comparable
- [x] Identifiable
- [ ] CustomStringConvertible
> Identifiable provides a stable `id` so SwiftUI can tell items apart.

Q: How do you add a new computed property to `Int`, a type you don't own?
- [ ] Subclass Int
- [x] Write an extension on Int
- [ ] It's impossible
- [ ] Use a protocol
> Extensions add functionality to existing types without modifying their source.
```
