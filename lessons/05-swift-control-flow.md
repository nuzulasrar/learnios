---
id: swift-control-flow
title: "Control Flow: if, switch & Loops"
part: 2
minutes: 25
summary: Make decisions and repeat work with if, guard, Swift's powerful switch, ranges, and loops.
---

:::goals
- `if` / `else`, and `if` and `switch` as expressions
- `switch`: exhaustive, with ranges, multiple values, tuples and `where`
- Ranges: `...`, `..<` and `stride`
- `for-in`, `while` and `repeat-while` loops, `break` and `continue`
:::

## `if` / `else`

```swift runnable
let temperature = 28

if temperature > 30 {
    print("It's hot!")
} else if temperature > 20 {
    print("It's warm.")
} else {
    print("It's cool.")
}
```

No parentheses are needed around the condition, but the braces are **always** required. Comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=`. Logical operators: `&&` (and), `||` (or), `!` (not).

### `if` as an expression

You can assign the result of an `if` directly, and there's also the classic ternary `?:` for short cases:

```swift runnable
let age = 17
let status = if age >= 18 { "adult" } else { "minor" }
let ticket = age < 12 ? "child" : "standard"
print(status, ticket)
```

## `switch`

Swift's `switch` is powerful and safe: it must be **exhaustive** (handle every possible value), and cases don't fall through to the next one, so there's no `break` needed.

```swift runnable
func describe(day: Int) -> String {
    switch day {
    case 1:
        return "Monday"
    case 2:
        return "Tuesday"
    case 3, 4:                  // several values
        return "Midweek"
    case 5...7:                 // a range
        return "Almost weekend"
    default:                    // everything else
        return "Not a valid day"
    }
}

for d in [1, 4, 6, 99] {
    print(describe(day: d))
}
```

### `switch` as an expression, with `where`

```swift runnable
func grade(_ score: Int) -> String {
    switch score {
    case 90...: "A"                 // 90 and above
    case 80..<90: "B"
    case 70..<80: "C"
    case let s where s < 0: "Invalid"
    default: "Keep going"
    }
}

for s in [95, 83, 71, 40, -5] {
    print(s, grade(s))
}
```

### Matching tuples

```swift runnable
let point = (x: 0, y: 5)

switch point {
case (0, 0):
    print("Origin")
case (0, _):                    // _ matches anything
    print("On the y-axis")
case (_, 0):
    print("On the x-axis")
case let (x, y) where x == y:
    print("On the diagonal")
default:
    print("Somewhere else")
}
```

You'll use `switch` constantly with **enums** (Lesson 8), for example to show a loading, content or error view.

## Ranges

```swift runnable
let closed = 1...5              // 1, 2, 3, 4, 5  (includes the end)
let halfOpen = 1..<5            // 1, 2, 3, 4     (excludes the end)
let fromThree = 3...            // one-sided: 3 and up

print(Array(closed))
print(Array(halfOpen))
print(closed.contains(3))       // true
print(fromThree.contains(100))  // true

for i in stride(from: 0, through: 20, by: 5) {   // 0, 5, 10, 15, 20
    print(i, terminator: " ")
}
print()
for i in stride(from: 10, to: 0, by: -3) {       // 10, 7, 4, 1
    print(i, terminator: " ")
}
print()
```

`..<` is perfect for indices: `for i in 0..<array.count`.

## `for-in` loops

```swift runnable
for i in 1...3 {
    print("Round \(i)")
}

let fruits = ["apple", "banana", "cherry"]
for fruit in fruits {
    print(fruit)
}

for (index, fruit) in fruits.enumerated() {     // index and value
    print("\(index): \(fruit)")
}

for _ in 1...3 {                                 // don't need the value
    print("🎉", terminator: "")
}
print()
```

## `while` and `repeat-while`

```swift runnable
var countdown = 3
while countdown > 0 {
    print("\(countdown)…")
    countdown -= 1
}
print("Liftoff! 🚀")

var attempts = 0
repeat {                 // runs at least once
    attempts += 1
    print("Attempt \(attempts)")
} while attempts < 2
```

Swift has no `++` or `--` operators. Use `+= 1` and `-= 1`.

## `break` and `continue`

```swift runnable
for n in 1...10 {
    if n % 2 == 0 { continue }   // skip even numbers
    if n > 7 { break }           // stop the loop
    print(n)
}
```

## Recap

:::recap
- `if` needs braces, not parentheses. `if` and `switch` can produce values.
- `switch` must be exhaustive and doesn't fall through. It matches values, ranges, tuples and `where` conditions.
- Ranges: `a...b` includes `b`, `a..<b` excludes it. `stride` counts by steps.
- `for-in` iterates anything iterable; `enumerated()` adds indices.
:::

## Practice

:::exercise
1. **FizzBuzz** for 1 to 20: print "Fizz" for multiples of 3, "Buzz" for multiples of 5, "FizzBuzz" for both, otherwise the number. Use a `switch` on the tuple `(n % 3, n % 5)`.
2. Write `func bmiCategory(_ bmi: Double) -> String` using `switch` with ranges: under 18.5 → "Underweight", 18.5 to under 25 → "Normal", 25 to under 30 → "Overweight", 30+ → "Obese".
3. Print the 7 times table (7 × 1 up to 7 × 10).
:::

:::solution Show solutions
```swift runnable
for n in 1...20 {
    switch (n % 3, n % 5) {
    case (0, 0): print("FizzBuzz")
    case (0, _): print("Fizz")
    case (_, 0): print("Buzz")
    default: print(n)
    }
}

func bmiCategory(_ bmi: Double) -> String {
    switch bmi {
    case ..<18.5: "Underweight"
    case 18.5..<25: "Normal"
    case 25..<30: "Overweight"
    default: "Obese"
    }
}
print(bmiCategory(22.4))

for i in 1...10 {
    print("7 × \(i) = \(7 * i)")
}
```
:::

## Check your understanding

```quiz
Q: What happens when a switch doesn't handle every possible value and has no default?
- [ ] Unmatched values are ignored
- [x] It's a compile error, because switch must be exhaustive
- [ ] It crashes at runtime
- [ ] It falls through to the first case
> Swift requires every possible value to be handled.

Q: Which range contains exactly 0, 1, 2, 3, 4?
- [ ] `0...5`
- [x] `0..<5`
- [ ] `1...4`
- [ ] `stride(from: 0, to: 4, by: 1)`
> The half-open range `..<` excludes its upper bound.

Q: How do you increase a variable by one in Swift?
- [ ] `count++`
- [x] `count += 1`
- [ ] `++count`
- [ ] `count.increment()`
> Swift removed ++ and -- to avoid confusion.

Q: What does `continue` do in a loop?
- [ ] Exits the loop
- [x] Skips to the next iteration
- [ ] Restarts the loop
- [ ] Pauses execution
> `break` exits; `continue` skips the rest of the current iteration.
```
