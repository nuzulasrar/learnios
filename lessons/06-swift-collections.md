---
id: swift-collections
title: Collections & Closures
part: 2
minutes: 35
summary: Arrays, sets and dictionaries, plus closures and the functional operations (map, filter, sorted, reduce and friends) you'll use in every screen.
---

:::goals
- Arrays, sets and dictionaries, and when to use each
- Why `let` collections can't change and `var` ones can
- Closure syntax, shorthand arguments (`$0`) and trailing closures
- `map`, `filter`, `sorted`, `reduce`, `compactMap`, `first(where:)`, `Dictionary(grouping:by:)` and more
- Tuples
:::

## Arrays

An **array** is an ordered list of values of the same type.

```swift runnable
var colors = ["red", "green", "blue"]      // [String]
print(colors[0])                           // "red"
print(colors.count)                        // 3
print(colors.first ?? "none")              // "red"
print(colors.contains("green"))            // true

colors.append("yellow")
colors.insert("purple", at: 0)
colors.remove(at: 1)                       // removes "red"
colors[0] = "violet"
print(colors)

let empty: [Int] = []                      // an empty array needs a type
print(empty.isEmpty)

let combined = [1, 2] + [3, 4]             // [1, 2, 3, 4]
print(combined)
```

:::tip `let` vs `var` for collections
A collection declared with `let` can't be changed at all: no appends, no removals. Arrays, sets and dictionaries are **value types** (Lesson 8), so copying one gives you an independent copy. Changing the copy never affects the original. This makes SwiftUI state predictable.
:::

:::danger Index out of range
`colors[10]` on a 4-element array **crashes** the app. Check `indices.contains(i)` first, or use `first`/`last`, which return optionals.
:::

## Sets

A **set** is an unordered collection of **unique** values, which makes "is this in there?" checks very fast.

```swift runnable
var tags: Set = ["swift", "ios", "swift"]
print(tags.count)                 // 2: duplicates are ignored
tags.insert("xcode")
print(tags.contains("ios"))       // true

let a: Set = [1, 2, 3, 4]
let b: Set = [3, 4, 5]
print(a.intersection(b).sorted()) // [3, 4]
print(a.union(b).sorted())        // [1, 2, 3, 4, 5]
print(a.subtracting(b).sorted())  // [1, 2]
```

## Dictionaries

A **dictionary** maps unique **keys** to **values**.

```swift runnable
var capitals = [
    "France": "Paris",
    "Japan": "Tokyo",
    "Kenya": "Nairobi"
]                                                   // [String: String]

print(capitals["Japan"] ?? "?")                     // Tokyo
print(capitals["Mars"] ?? "Unknown")                // lookups return optionals
print(capitals["Mars", default: "Unknown"])         // or give a default

capitals["Peru"] = "Lima"                           // add
capitals["Kenya"] = nil                             // remove

for (country, city) in capitals.sorted(by: { $0.key < $1.key }) {
    print("\(city) is the capital of \(country)")
}

var stock = ["apples": 5]
stock["apples", default: 0] += 1                    // update with a default
stock["pears", default: 0] += 3
print(stock.sorted { $0.key < $1.key })
```

Dictionaries (and sets) are **unordered**, so sort them when you need a stable order.

## Closures

A **closure** is a block of code you can store and pass around, like a function without a name. The full syntax:

```swift runnable
let double = { (x: Int) -> Int in
    return x * 2
}
print(double(21))

let sayHello = { print("Hello!") }     // no parameters
sayHello()
```

Parameters and return type go before `in`, and the body after it.

### Getting shorter

Swift lets you drop anything it can infer. These all do the same thing:

```swift runnable
let numbers = [5, 2, 8, 1]

let a = numbers.sorted(by: { (x: Int, y: Int) -> Bool in return x < y })
let b = numbers.sorted(by: { x, y in x < y })   // types inferred, implicit return
let c = numbers.sorted(by: { $0 < $1 })         // shorthand argument names
let d = numbers.sorted { $0 < $1 }              // trailing closure
let e = numbers.sorted(by: <)                   // an operator is a function too!

print(a, b, c, d, e)
```

- **`$0`, `$1`** refer to the first and second arguments.
- **Trailing closures**: if the last argument is a closure, write it after the parentheses, and drop the parentheses if it's the only argument.

:::note This is exactly how SwiftUI looks
`VStack { Text("Hi") }` and `Button("Save") { save() }` are just functions and initialisers taking trailing closures. Once you're comfortable with closures, SwiftUI code reads naturally.
:::

## Functional operations

These methods take closures and return **new** collections. They're the bread and butter of Swift.

```swift runnable
struct Product {
    let name: String
    let price: Double
    let category: String
    let inStock: Bool
}

let products = [
    Product(name: "Laptop", price: 1200, category: "Tech", inStock: true),
    Product(name: "Mouse", price: 25, category: "Tech", inStock: true),
    Product(name: "Desk", price: 300, category: "Furniture", inStock: false),
    Product(name: "Chair", price: 150, category: "Furniture", inStock: true),
    Product(name: "Monitor", price: 280, category: "Tech", inStock: false)
]

// filter: keep matching items
print(products.filter { $0.inStock }.map(\.name))

// map: transform each item
print(products.map { $0.name.uppercased() })

// sorted(by:)
print(products.sorted { $0.price < $1.price }.map(\.name))

// first(where:) returns an optional
print(products.first { $0.price > 500 }?.name ?? "none")

// contains(where:), allSatisfy
print(products.contains { !$0.inStock })
print(products.allSatisfy { $0.price > 10 })

// reduce: combine into one value
let total = products.reduce(0) { $0 + $1.price }
print(total)

// max(by:), min(by:)
print(products.max { $0.price < $1.price }?.name ?? "")

// Dictionary(grouping:by:): [Key: [Item]]
let byCategory = Dictionary(grouping: products, by: \.category)
for (category, items) in byCategory.sorted(by: { $0.key < $1.key }) {
    print(category, items.count)
}

// compactMap: transform and drop nils
let inputs = ["4", "x", "15", "", "8"]
print(inputs.compactMap { Int($0) })        // [4, 15, 8]
```

`\.name` is a **key path**, a reference to a property. `map(\.name)` is shorthand for `map { $0.name }`. SwiftUI uses key paths a lot, e.g. `ForEach(items, id: \.id)`.

### Chaining

```swift runnable
import Foundation   // for .capitalized

let words = ["swift", "is", "a", "fast", "and", "expressive", "language"]

let result = words
    .filter { $0.count > 3 }
    .map { $0.capitalized }
    .sorted { $0.count < $1.count }
    .joined(separator: " · ")

print(result)
```

:::note Foundation
Some handy string features, like `capitalized`, come from **Foundation**, Apple's base library of dates, URLs, formatting, JSON and more. Every iOS app has it: `import SwiftUI` imports Foundation for you. In a standalone Swift file you add `import Foundation` yourself.
:::

## Tuples

A **tuple** groups a few values without defining a type, which is handy for returning multiple values:

```swift runnable
func minMax(of values: [Int]) -> (min: Int, max: Int)? {
    guard let first = values.first else { return nil }
    var lo = first, hi = first
    for v in values { lo = min(lo, v); hi = max(hi, v) }
    return (lo, hi)
}

if let range = minMax(of: [8, 3, 12, 5]) {
    print("From \(range.min) to \(range.max)")
}

let (name, age) = ("Ada", 36)     // destructuring
print(name, age)
```

For anything more than a quick return value, prefer a `struct` (Lesson 8).

## Recap

:::recap
- **Array** (ordered), **Set** (unique, fast lookup), **Dictionary** (key → value). Lookups that can fail return optionals.
- Collections are value types; `let` makes them fully immutable.
- Closures: `{ params in body }`, shortened with `$0` and trailing closure syntax.
- `map`, `filter`, `sorted`, `reduce`, `compactMap`, `first(where:)`: chain them for readable data pipelines.
:::

## Practice

:::exercise
Given:

```swift
struct Student { let name: String; let grade: Int; let house: String }

let students = [
    Student(name: "Amara", grade: 88, house: "Red"), Student(name: "Ben", grade: 72, house: "Blue"),
    Student(name: "Chen", grade: 95, house: "Red"), Student(name: "Dara", grade: 64, house: "Green"),
    Student(name: "Eli", grade: 79, house: "Blue"), Student(name: "Fay", grade: 91, house: "Green")
]
```

1. Print the names of students with a grade of 80 or more, sorted alphabetically.
2. Print the average grade.
3. Print each house with its best student.
4. Print whether any student failed (below 65).
:::

:::solution Show solution
```swift runnable
struct Student { let name: String; let grade: Int; let house: String }

let students = [
    Student(name: "Amara", grade: 88, house: "Red"), Student(name: "Ben", grade: 72, house: "Blue"),
    Student(name: "Chen", grade: 95, house: "Red"), Student(name: "Dara", grade: 64, house: "Green"),
    Student(name: "Eli", grade: 79, house: "Blue"), Student(name: "Fay", grade: 91, house: "Green")
]

// 1
print(students.filter { $0.grade >= 80 }.map(\.name).sorted())

// 2
let average = Double(students.reduce(0) { $0 + $1.grade }) / Double(students.count)
print(average)

// 3
let houses = Dictionary(grouping: students, by: \.house)
for (house, members) in houses.sorted(by: { $0.key < $1.key }) {
    let best = members.max { $0.grade < $1.grade }!
    print("\(house): \(best.name)")
}

// 4
print(students.contains { $0.grade < 65 })
```
:::

## Check your understanding

```quiz
Q: What does `[1, 2, 3, 4].filter { $0 % 2 == 0 }.map { $0 * 10 }` return?
- [ ] [10, 20, 30, 40]
- [x] [20, 40]
- [ ] [2, 4]
- [ ] [10, 30]
> filter keeps 2 and 4; map multiplies each by 10.

Q: What is the type of `capitals["Japan"]` for a `[String: String]` dictionary?
- [ ] String
- [x] String? (the key might not exist)
- [ ] [String]
- [ ] Bool
> Dictionary lookups return optionals.

Q: What does `$0` mean inside a closure?
- [ ] The closure's return value
- [x] The closure's first argument
- [ ] The first element of the array
- [ ] A global variable
> Shorthand argument names $0, $1, … refer to the closure's parameters in order.

Q: What does `["1", "a", "3"].compactMap { Int($0) }` return?
- [ ] [1, nil, 3]
- [x] [1, 3]
- [ ] ["1", "3"]
- [ ] It crashes
> compactMap transforms each element and drops the nil results.
```
