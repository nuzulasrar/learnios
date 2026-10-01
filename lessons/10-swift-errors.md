---
id: swift-errors
title: Error Handling
part: 2
minutes: 20
summary: Things go wrong, like no internet, bad input or missing files. Learn Swift's explicit error handling with throws, do/catch, try?, custom errors and Result.
---

:::goals
- Defining errors with enums that conform to `Error`
- Throwing functions and `do` / `try` / `catch`
- `try?` and `try!`
- Typed throws
- `defer` for cleanup, and `Result`
:::

## Defining errors

An error is any type conforming to the `Error` protocol. Enums are ideal because they list everything that can go wrong:

```swift
enum LoginError: Error {
    case emptyPassword
    case tooShort(minimum: Int)
    case accountLocked
}
```

## Throwing functions

A function that can fail is marked **`throws`**. It uses **`throw`** to report an error, which immediately exits the function:

```swift runnable
enum LoginError: Error {
    case emptyPassword
    case tooShort(minimum: Int)
}

func validate(password: String) throws -> Bool {
    if password.isEmpty { throw LoginError.emptyPassword }
    if password.count < 8 { throw LoginError.tooShort(minimum: 8) }
    return true
}

do {
    try validate(password: "swift")              // `try` marks the call that can throw
    print("Password OK")                         // skipped if it threw
} catch LoginError.emptyPassword {
    print("Please enter a password.")
} catch LoginError.tooShort(let minimum) {
    print("Password must be at least \(minimum) characters.")
} catch {
    print("Unexpected error: \(error)")          // `error` is available automatically
}
```

Swift makes errors **visible**: you can't call a throwing function without writing `try`, and you must handle the error or pass it on (by marking your own function `throws`).

## `try?` and `try!`

```swift runnable
enum ParseError: Error { case notANumber }

func parse(_ text: String) throws -> Int {
    guard let n = Int(text) else { throw ParseError.notANumber }
    return n
}

let a = try? parse("42")      // Optional(42)
let b = try? parse("x")       // nil: the error is discarded
print(a ?? -1, b ?? -1)

let c = try! parse("7")       // crashes if it throws, so only use when failure is impossible
print(c)
```

- **`try?`** turns the result into an optional: `nil` if it threw. It's convenient when you don't care *why* it failed.
- **`try!`** crashes on error. Avoid it, just like `!` for optionals.

## User-friendly messages

Conform to `LocalizedError` to give each error a message you can show in an alert:

```swift runnable
import Foundation

enum NetworkError: LocalizedError {
    case offline
    case server(code: Int)

    var errorDescription: String? {
        switch self {
        case .offline: "You're offline. Check your connection and try again."
        case .server(let code): "The server had a problem (\(code)). Please try later."
        }
    }
}

let error: Error = NetworkError.server(code: 503)
print(error.localizedDescription)
```

## Typed throws

By default, a throwing function can throw *any* `Error`. Swift 6 lets you declare exactly which type, so callers get that specific type in `catch`:

```swift runnable
enum ValidationError: Error { case empty, tooLong(max: Int) }

func validateName(_ name: String) throws(ValidationError) -> String {
    if name.isEmpty { throw .empty }
    if name.count > 20 { throw .tooLong(max: 20) }
    return name
}

do {
    _ = try validateName("")
} catch {                                 // error is a ValidationError here, not `any Error`
    switch error {
    case .empty: print("Name can't be empty")
    case .tooLong(let max): print("Max \(max) characters")
    }
}
```

Typed throws are handy for small, closed sets of errors. For most app code, plain `throws` is fine.

## `defer`: always clean up

A `defer` block runs when the current scope exits, however it exits (return, throw or reaching the end):

```swift runnable
func process() throws {
    print("Start loading")
    defer { print("Stop loading spinner") }   // runs last, even if we throw
    throw URLErrorLike.timeout
}
enum URLErrorLike: Error { case timeout }

do { try process() } catch { print("Caught: \(error)") }
```

You'll use `defer { isLoading = false }` in view models so the spinner always stops.

## `Result`

`Result<Success, Failure>` stores either a value or an error. It's useful when you need to keep an outcome and look at it later:

```swift runnable
enum MathError: Error { case divideByZero }

func divide(_ a: Int, by b: Int) -> Result<Int, MathError> {
    b == 0 ? .failure(.divideByZero) : .success(a / b)
}

for result in [divide(10, by: 2), divide(1, by: 0)] {
    switch result {
    case .success(let value): print("Answer: \(value)")
    case .failure(let error): print("Oops: \(error)")
    }
}
```

With `async`/`await` (next lesson), plain `throws` is usually cleaner than `Result`.

## Recap

:::recap
- Errors conform to `Error`, and enums are ideal.
- `throws` + `throw` to fail; call with `try` inside `do { } catch { }`.
- `try?` gives an optional; `try!` crashes, so avoid it.
- `LocalizedError` provides user-facing messages; `defer` guarantees cleanup.
:::

:::exercise
Write `func withdraw(_ amount: Double, from balance: Double) throws -> Double` that throws `BankError.invalidAmount` for amounts ≤ 0 and `BankError.insufficientFunds(needed: Double)` when the balance is too low. Make `BankError` a `LocalizedError` with friendly messages, and call it for three cases inside `do/catch`, printing `error.localizedDescription`.
:::

:::solution Show solution
```swift runnable
import Foundation

enum BankError: LocalizedError {
    case invalidAmount
    case insufficientFunds(needed: Double)

    var errorDescription: String? {
        switch self {
        case .invalidAmount: "Enter an amount greater than zero."
        case .insufficientFunds(let needed): "You need \(needed) more to do that."
        }
    }
}

func withdraw(_ amount: Double, from balance: Double) throws -> Double {
    guard amount > 0 else { throw BankError.invalidAmount }
    guard amount <= balance else { throw BankError.insufficientFunds(needed: amount - balance) }
    return balance - amount
}

for amount in [30.0, -5, 500] {
    do {
        let newBalance = try withdraw(amount, from: 100)
        print("New balance: \(newBalance)")
    } catch {
        print(error.localizedDescription)
    }
}
```
:::

## Check your understanding

```quiz
Q: What keyword must you write before calling a throwing function?
- [ ] throw
- [x] try
- [ ] catch
- [ ] await
> `try` marks every call that can throw, so errors are always visible in the code.

Q: What does `try? parse("x")` return if parse throws?
- [ ] It crashes
- [x] nil
- [ ] An empty string
- [ ] The error
> try? converts errors into nil.

Q: When does a `defer` block run?
- [ ] Immediately
- [ ] Only if an error is thrown
- [x] When the enclosing scope exits, however it exits
- [ ] At app launch
> defer is perfect for cleanup that must always happen.
```
