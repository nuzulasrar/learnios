---
id: swift-basics
title: "Swift Basics: Constants, Types & Functions"
part: 2
minutes: 30
summary: Start writing Swift, the language of every Apple platform. Constants and variables, basic types, strings, and functions with Swift's distinctive argument labels.
---

:::goals
- The difference between `let` and `var` (and why you should prefer `let`)
- Swift's basic types and type inference
- String interpolation and multi-line strings
- Functions, including **argument labels**, default values and `inout`
:::

## Why learn Swift before iOS?

Every Apple API you'll use is called from Swift, and SwiftUI leans on Swift features like closures, protocols and property wrappers. Spending a few lessons on pure Swift first will make Parts 3–8 far smoother.

:::tip Run the examples
Press **▶ Try online** on any example: it copies the code and opens [SwiftFiddle](https://swiftfiddle.com), where you paste and run it. You can also use an **Xcode Playground** (**File → New → Playground → Blank**), which runs Swift code as you type.
:::

## Your first Swift program

```swift runnable
print("Hello, Swift!")
```

That's a complete program. In a playground or script, top-level code simply runs from top to bottom. No `main` function or semicolons needed.

## Constants and variables: `let` and `var`

```swift runnable
let name = "Ada"        // let = constant (can't be changed)
var score = 10          // var = variable (can be changed)

score = score + 5       // ✅ fine
score += 1              // ✅ shorthand
// name = "Grace"       // ❌ error: cannot assign to value: 'name' is a 'let' constant

print(name)
print(score)
```

:::tip Prefer `let`
Use `let` by default and switch to `var` only when the value really needs to change. Xcode even warns you: *"Variable was never mutated; consider changing to 'let'"*. Code where values don't change unexpectedly is easier to reason about, and it's safer with concurrency.
:::

## Basic types

Swift is **type-safe**: every value has a type known at compile time. You rarely write it, because Swift **infers** it:

```swift runnable
let age = 30                 // Int
let price = 19.99            // Double
let isMember = true          // Bool
let initial: Character = "K" // Character (needs the annotation)
let city = "Lagos"           // String
let population = 15_000_000  // underscores make big numbers readable

// You can also write the type explicitly:
let temperature: Double = 21
let count: Int = 3

print(age, price, isMember, initial, city, population, temperature, count)
```

| Type | Holds | Example |
|---|---|---|
| `Int` | Whole numbers (64-bit on modern devices) | `42` |
| `Double` | Decimal numbers (the default for decimals) | `3.14` |
| `Float` | Less precise decimals (graphics) | `Float(3.14)` |
| `Bool` | `true` or `false` | `true` |
| `String` | Text | `"Hello"` |
| `Character` | A single character | `"A"` |

### Converting between types

Swift never converts numbers implicitly. Create a new value of the type you want:

```swift runnable
let whole = 7
let decimal = Double(whole)      // 7.0
let text = String(whole)         // "7"
let parsed = Int("123")          // Optional(123): parsing can fail
let failed = Int("abc")          // nil

print(decimal)
print(text + "!")
print(parsed ?? 0)               // ?? gives a fallback (Lesson 7)
print(failed ?? 0)
print(7 / 2)                     // 3   (integer division!)
print(7.0 / 2)                   // 3.5
print(7 % 2)                     // 1   (remainder)
// print(whole + 1.5)            // ❌ error: can't add Int and Double
print(Double(whole) + 1.5)       // ✅ 8.5
```

:::danger Integer division
`7 / 2` is `3`, because both numbers are `Int`. If you need decimals, make both sides `Double`: `Double(7) / 2`.
:::

## Strings

### String interpolation

Insert any value into a string with `\( )`:

```swift runnable
let name = "Sam"
let items = 3
let price = 4.5

print("Hi \(name)!")
print("You have \(items) items")
print("Total: \(Double(items) * price)")
print("Name has \(name.count) letters")
print("Shouting: \(name.uppercased())")
```

### Useful string operations

```swift runnable
let text = "Swift Development"
print(text.count)                       // 17
print(text.lowercased())                // "swift development"
print(text.contains("Dev"))             // true
print(text.hasPrefix("Swift"))          // true
print(text.split(separator: " "))       // ["Swift", "Development"]
print(text.replacing(" ", with: "_"))   // "Swift_Development"
print(text.isEmpty)                     // false
print(text.first!)                      // "S"
print(String(text.reversed()))          // "tnempoleveD tfiwS"

var greeting = "Hello"
greeting += ", world"                   // strings declared with var can change
print(greeting)
```

### Multi-line strings

```swift runnable
let poem = """
    Roses are red,
    Swift is so neat,
    Optionals make
    my code complete.
    """
print(poem)
```

The indentation of the closing `"""` is removed from every line, so you can indent the string along with your code.

## Functions

```swift runnable
func greet(person: String) -> String {
    return "Hello, \(person)!"
}

print(greet(person: "Ada"))
```

- `func` declares a function; parameters have a name and type; `-> Type` is the return type.
- If the body is a **single expression**, you can omit `return`:

```swift
func square(_ x: Int) -> Int { x * x }
```

### Argument labels ⭐

Swift functions read like sentences because each parameter can have an **argument label** (used when calling) separate from its **parameter name** (used inside):

```swift runnable
// label  name
func move(from start: String, to end: String) -> String {
    "Moving from \(start) to \(end)"
}
print(move(from: "Paris", to: "Rome"))

// Use _ to omit the label when it reads naturally without one
func square(_ number: Int) -> Int {
    number * number
}
print(square(9))

// No label written = the parameter name is the label
func area(width: Double, height: Double) -> Double {
    width * height
}
print(area(width: 3, height: 4))
```

:::note You'll see this everywhere
Apple's APIs are designed around labels: `Image(systemName: "star")`, `list.insert(item, at: 0)`, `.padding(.horizontal, 16)`. Reading them aloud tells you what they do.
:::

### Default values

```swift runnable
func orderCoffee(size: String = "Medium", sugar: Int = 0, milk: Bool = true) -> String {
    "\(size) coffee, \(sugar) sugar, milk: \(milk)"
}

print(orderCoffee())
print(orderCoffee(size: "Large"))
print(orderCoffee(sugar: 2))
print(orderCoffee(size: "Small", milk: false))
```

Arguments must appear in the same order as the parameters, but you can skip any that have defaults.

### Variadic and `inout` parameters

```swift runnable
func average(_ numbers: Double...) -> Double {   // variadic: any number of values
    guard !numbers.isEmpty else { return 0 }
    return numbers.reduce(0, +) / Double(numbers.count)
}
print(average(4, 8, 6))

func doubleInPlace(_ value: inout Int) {          // inout: can modify the caller's variable
    value *= 2
}
var points = 21
doubleInPlace(&points)                            // & marks it as modifiable
print(points)
```

Parameters are constants by default. `inout` is the explicit exception.

## Comments

```swift
// Single-line comment

/*
 Multi-line comment
 */

/// Documentation comment: shown in Xcode's Quick Help (Option-click).
/// - Parameter name: The person to greet.
func greet(_ name: String) { print("Hi \(name)") }
```

## Recap

:::recap
- `let` for constants, `var` for variables. Prefer `let`.
- Types are inferred but strict; convert explicitly with `Double(x)`, `String(x)`, `Int("42")`.
- String interpolation: `"\(value)"`.
- Functions have **argument labels**; use `_` to omit them and give defaults with `= value`.
:::

## Practice

:::exercise
1. Write `func celsiusToFahrenheit(_ c: Double) -> Double` (formula: `c * 9 / 5 + 32`) and print the result for 0, 37 and 100.
2. Write `func describe(name: String, age: Int, city: String = "Unknown") -> String` that returns e.g. *"Ada is 36 and lives in London."* Call it with and without `city`.
3. Write `func initials(of first: String, and last: String) -> String` that returns e.g. `"A.L."`. Call it as `initials(of: "Ada", and: "Lovelace")`.
:::

:::solution Show solutions
```swift runnable
func celsiusToFahrenheit(_ c: Double) -> Double {
    c * 9 / 5 + 32
}

func describe(name: String, age: Int, city: String = "Unknown") -> String {
    "\(name) is \(age) and lives in \(city)."
}

func initials(of first: String, and last: String) -> String {
    "\(first.prefix(1)).\(last.prefix(1))."
}

print(celsiusToFahrenheit(0))
print(celsiusToFahrenheit(37))
print(celsiusToFahrenheit(100))
print(describe(name: "Ada", age: 36, city: "London"))
print(describe(name: "Alan", age: 41))
print(initials(of: "Ada", and: "Lovelace"))
```
:::

## Check your understanding

```quiz
Q: What happens with `let x = 5` followed by `x = 6`?
- [ ] x becomes 6
- [x] Compile error: x is a constant
- [ ] Runtime crash
- [ ] x becomes 11
> Constants declared with `let` can't be reassigned.

Q: What is the type of `Int("42")`?
- [ ] Int
- [x] Int? (an optional Int), because the conversion can fail
- [ ] String
- [ ] Double
> Parsing a string may not produce a number, so the result is optional (Lesson 7).

Q: Given `func move(from start: String, to end: String)`, how do you call it?
- [ ] `move(start: "A", end: "B")`
- [x] `move(from: "A", to: "B")`
- [ ] `move("A", "B")`
- [ ] `move(from start: "A", to end: "B")`
> Callers use argument labels; the parameter names are used inside the function.

Q: How do you insert a value into a string?
- [ ] `"Total: $total"`
- [ ] `"Total: {total}"`
- [x] `"Total: \(total)"`
- [ ] `"Total: " + total` for any type
> Swift uses backslash-parentheses for interpolation.
```
