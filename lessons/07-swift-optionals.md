---
id: swift-optionals
title: Optionals
part: 2
minutes: 25
summary: Swift's answer to "no value". Optionals make missing data explicit and remove a whole class of crashes. Learn if let, guard let, ??, optional chaining, and why ! is dangerous.
---

:::goals
- What an optional is and why Swift has them
- Unwrapping safely with `if let` and `guard let`
- The nil-coalescing operator `??`
- Optional chaining with `?.`
- Why force-unwrapping `!` is dangerous
:::

## The problem: missing values

Sometimes a value might not exist: a user without a nickname, a number that failed to parse, a search with no result. Many languages let *any* variable secretly be `null`, and using it crashes the app.

Swift makes this **explicit in the type**. A regular `String` *always* contains a string. A `String?` (an **optional** String) contains either a string **or `nil`**:

```swift runnable
var nickname: String? = "Addy"
print(nickname as Any)      // Optional("Addy")
nickname = nil              // ✅ allowed for optionals

var name: String = "Ada"
// name = nil               // ❌ error: 'nil' cannot be assigned to type 'String'
print(name)
```

You can't use an optional as if it were a plain value. The compiler makes you handle the `nil` case first:

```swift
let length = nickname.count   // ❌ error: value of optional type 'String?' must be unwrapped
```

That turns a potential crash into a compile-time error you fix before shipping. 🎉

:::analogy
An optional is a gift box. It might contain something, or it might be empty. Before you can use what's inside, you have to open the box and check.
:::

## `if let`: unwrap when present

```swift runnable
let input = "42"

if let number = Int(input) {             // Int(input) is Int?
    print("Got a number: \(number)")     // number is a plain Int here
} else {
    print("Not a number")
}

let nickname: String? = "Addy"
if let nickname {                        // shorthand: unwrap into the same name
    print("Hi, \(nickname)!")
}

// Unwrap several at once, with an extra condition:
let a: Int? = 4, b: Int? = 6
if let a, let b, a < b {
    print("\(a) < \(b)")
}
```

## `guard let`: exit early

`guard` checks a condition and requires you to **leave the scope** (usually `return`) if it fails. The unwrapped value stays available for the rest of the function, which keeps the "happy path" unindented:

```swift runnable
func greet(_ name: String?) {
    guard let name, !name.isEmpty else {
        print("Hello, stranger!")
        return
    }
    print("Hello, \(name)!")    // name is a non-optional String from here on
}

greet("Ada")
greet(nil)
greet("")
```

:::tip `if let` or `guard let`?
Use `guard` for **preconditions**: "if I don't have this, I can't continue." Use `if let` when you do something extra only if the value exists. You'll see `guard` a lot at the top of functions.
:::

## Nil-coalescing: `??`

`a ?? b` means "use `a` if it has a value, otherwise `b`":

```swift runnable
let input: String? = nil
let name = input ?? "Guest"
print("Welcome, \(name)")

let scores = ["Ada": 91]
print(scores["Grace"] ?? 0)
print(Int("abc") ?? -1)
```

## Optional chaining: `?.`

Call properties and methods on an optional. If it's `nil`, the whole expression becomes `nil` instead of crashing:

```swift runnable
struct Address { var city: String }
struct User { var name: String; var address: Address? }

let ada = User(name: "Ada", address: Address(city: "London"))
let bob = User(name: "Bob", address: nil)

print(ada.address?.city ?? "no city")                 // London
print(bob.address?.city ?? "no city")                 // no city
print(bob.address?.city.uppercased() ?? "-")          // chains stop at the first nil

let users: [User] = [ada, bob]
print(users.first?.name ?? "empty")
```

## Force unwrapping: `!` (avoid!)

`value!` says "I'm *certain* this isn't nil, so give me the value." If you're wrong, **the app crashes**:

```swift
let number = Int("abc")!     // 💥 Fatal error: Unexpectedly found nil while unwrapping an Optional value
```

:::danger Treat `!` as a code smell
Every `!` is a potential crash. Almost always there's a safer option: `if let`, `guard let`, `??` or optional chaining. The rare acceptable cases are values that can *only* be nil because of a programmer mistake, like a hard-coded URL you know is valid: `URL(string: "https://apple.com")!`.
:::

## Optionals and collections

```swift runnable
let values = [3, 8, 1]
print(values.first as Any)        // Optional(3)
print(values.max() ?? 0)          // 8
print([Int]().max() ?? 0)         // 0: empty array, so max() is nil

let maybeNumbers = ["1", "two", "3"].map { Int($0) }   // [Int?]
print(maybeNumbers)
print(maybeNumbers.compactMap { $0 })                   // [1, 3]
```

Note the difference: `[Int?]` is an array that may *contain* nils; `[Int]?` is an array that may itself *be* nil.

## Optional `map`

Transform an optional's value only if it exists:

```swift runnable
let text: String? = "42"
let doubled = text.flatMap { Int($0) }.map { $0 * 2 }
print(doubled ?? 0)   // 84
```

## Recap

:::recap
- `T?` means "a `T` or `nil`". Non-optional types can never be nil.
- Unwrap with `if let x` / `guard let x else { return }`.
- `??` supplies a fallback; `?.` chains safely through optionals.
- `!` crashes on nil, so avoid it.
:::

## Practice

:::exercise
1. Write `func lengthOrZero(_ s: String?) -> Int` using optional chaining and `??`.
2. Write `func parseAge(_ input: String?) -> String` that returns `"Age: N"` for a valid number, or `"Invalid age"` otherwise. Use `guard let`.
3. Given `let users = [1: "Ada", 2: "Linus"]`, print the uppercased name for id 3, falling back to `"UNKNOWN"`, in one line.
:::

:::solution Show solutions
```swift runnable
func lengthOrZero(_ s: String?) -> Int {
    s?.count ?? 0
}

func parseAge(_ input: String?) -> String {
    guard let input, let age = Int(input) else {
        return "Invalid age"
    }
    return "Age: \(age)"
}

print(lengthOrZero(nil))
print(lengthOrZero("hey"))
print(parseAge("42"))
print(parseAge("forty"))
print(parseAge(nil))

let users = [1: "Ada", 2: "Linus"]
print(users[3]?.uppercased() ?? "UNKNOWN")
```
:::

## Check your understanding

```quiz
Q: Which declaration can hold nil?
- [ ] `let a: String = nil`
- [x] `let a: String? = nil`
- [ ] `let a: String! = nil` is the only way
- [ ] `let a = nil`
> Only optional types can hold nil. `let a = nil` doesn't compile because there's no type to infer.

Q: What does `let x = s?.count ?? -1` give when `s` is nil?
- [ ] nil
- [ ] 0
- [x] -1
- [ ] It crashes
> The optional chain produces nil, so ?? returns the fallback.

Q: What must the `else` branch of a `guard` statement do?
- [ ] Print an error
- [x] Exit the current scope, e.g. with return, throw, break or continue
- [ ] Nothing special
- [ ] Unwrap the value
> guard requires you to leave the scope if the condition fails, which is what makes the unwrapped value safe afterwards.

Q: Why should you avoid `!` force unwrapping?
- [ ] It's slower
- [x] It crashes the app if the value is nil
- [ ] It's deprecated
- [ ] It makes the value optional
> Force unwrapping bypasses the safety optionals provide.
```
