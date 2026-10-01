---
id: testing
title: Testing with Swift Testing & XCTest
part: 7
minutes: 40
summary: Ship with confidence. Write fast unit tests with Swift Testing, test async view models with mocks, use parameterized tests, and automate the UI with XCTest.
---

:::goals
- Why test, and what to test first
- Swift Testing: `@Test`, `#expect`, `#require`, `@Suite`
- Parameterized tests
- Testing async view models with mock services
- UI tests with XCUITest
- Running tests and reading results
:::

## Why test?

Every change you make might break something that used to work. Automated tests check it for you in seconds, so you can refactor fearlessly and catch bugs before your users do.

<div class="diagram"><div class="stack">
<div>UI tests<small>few · slow · drive the real app in the Simulator</small></div>
<div>Integration tests<small>some · several real parts together</small></div>
<div class="hl">Unit tests<small>many · fast · one type in isolation</small></div>
</div></div>

Your project already has two test targets: **`HelloiOSTests`** for unit tests (Swift Testing) and **`HelloiOSUITests`** for UI tests (XCTest).

## Swift Testing basics

**Swift Testing** is Apple's modern testing framework. Say you have this function in your app:

```swift title="Formatting.swift"
func formatMinutes(_ total: Int) -> String {
    precondition(total >= 0, "Minutes can't be negative")
    let h = total / 60, m = total % 60
    switch (h, m) {
    case (0, _): return "\(m) min"
    case (_, 0): return "\(h) h"
    default: return "\(h) h \(m) min"
    }
}
```

Tests are functions marked **`@Test`**, with **`#expect`** for checks:

```swift title="FormattingTests.swift" nocheck
import Testing
@testable import RecipeBox          // gives tests access to your app's internal types

struct FormattingTests {
    @Test func minutesUnderAnHour() {
        #expect(formatMinutes(45) == "45 min")
    }

    @Test func exactHours() {
        #expect(formatMinutes(120) == "2 h")
    }

    @Test("Mixed hours and minutes")              // a readable display name
    func mixed() {
        #expect(formatMinutes(75) == "1 h 15 min")
    }
}
```

- Put tests in any type (usually a struct), one test per behaviour.
- `#expect(condition)` records a failure with a helpful message showing the actual values, and the test continues.
- **`try #require(...)`** stops the test if it fails. Use it to unwrap optionals: `let user = try #require(users.first)`.
- Run tests with **[[Cmd]] + [[U]]**, or click the ◇ diamond next to a single test.

:::tip Arrange, Act, Assert
Structure each test in three parts: **arrange** the inputs, **act** by calling the code, **assert** with `#expect`. One behaviour per test, with a name that says what should happen.
:::

## Parameterized tests

Run the same test with many inputs, and each one is reported separately:

```swift nocheck
@Test("Formats minutes", arguments: [
    (5, "5 min"),
    (60, "1 h"),
    (61, "1 h 1 min"),
    (135, "2 h 15 min")
])
func formats(input: Int, expected: String) {
    #expect(formatMinutes(input) == expected)
}
```

## Testing a view model with a mock

This is where the architecture from Lesson 22 pays off. The view model depends on a **protocol**, so tests can inject a mock with controlled behaviour, with no network and no flakiness:

```swift title="RecipeListViewModel.swift (app)" nocheck
@MainActor
@Observable
final class RecipeListViewModel {
    private let service: any RecipeService
    private(set) var recipes: [Recipe] = []
    private(set) var errorMessage: String?
    var searchText = ""

    var visibleRecipes: [Recipe] {
        searchText.isEmpty ? recipes : recipes.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    init(service: any RecipeService) { self.service = service }

    func load() async {
        do {
            recipes = try await service.fetchRecipes()
            errorMessage = nil
        } catch {
            errorMessage = "Couldn't load recipes."
        }
    }
}
```

```swift title="RecipeListViewModelTests.swift" nocheck
import Testing
import Foundation
@testable import RecipeBox

struct MockRecipeService: RecipeService {
    var result: Result<[Recipe], Error>
    func fetchRecipes() async throws -> [Recipe] { try result.get() }
}

@MainActor
@Suite("Recipe list view model")
struct RecipeListViewModelTests {
    let sample = [
        Recipe(id: 1, name: "Margherita Pizza", cuisine: "Italian"),
        Recipe(id: 2, name: "Pad Thai", cuisine: "Thai")
    ]

    @Test func loadsRecipes() async {
        let vm = RecipeListViewModel(service: MockRecipeService(result: .success(sample)))

        await vm.load()

        #expect(vm.recipes == sample)
        #expect(vm.errorMessage == nil)
    }

    @Test func showsErrorWhenLoadingFails() async {
        let vm = RecipeListViewModel(service: MockRecipeService(result: .failure(URLError(.notConnectedToInternet))))

        await vm.load()

        #expect(vm.recipes.isEmpty)
        #expect(vm.errorMessage != nil)
    }

    @Test func searchFiltersByName() async {
        let vm = RecipeListViewModel(service: MockRecipeService(result: .success(sample)))
        await vm.load()

        vm.searchText = "thai"

        #expect(vm.visibleRecipes.map(\.name) == ["Pad Thai"])
    }
}
```

- Test functions can be `async` and `throws`. Just `await` the code under test.
- `@MainActor` on the suite lets tests call main-actor view models directly.
- `@Suite` groups related tests and gives them a name in the test navigator.

## Testing Codable decoding

Decoding bugs are common and easy to test. Include a sample JSON string:

```swift nocheck
@Test func decodesRecipeJSON() throws {
    let json = #"{"id": 7, "name": "Tacos", "cuisine": "Mexican", "extra": "ignored"}"#
    let recipe = try JSONDecoder().decode(Recipe.self, from: Data(json.utf8))
    #expect(recipe.name == "Tacos")
    #expect(recipe.cuisine == "Mexican")
}
```

## UI tests with XCTest

UI tests launch your **real app** in the Simulator and interact with it like a user, finding elements by their accessibility labels:

```swift title="RecipeBoxUITests.swift" nocheck
import XCTest

final class RecipeBoxUITests: XCTestCase {
    @MainActor
    func testAddingANote() throws {
        let app = XCUIApplication()
        app.launchArguments = ["-ui-testing"]          // let the app use mock data
        app.launch()

        let field = app.textFields["New note"]
        XCTAssertTrue(field.waitForExistence(timeout: 5))
        field.tap()
        field.typeText("Buy basil")
        app.buttons["Add"].tap()

        XCTAssertTrue(app.staticTexts["Buy basil"].exists)
    }
}
```

In the app, check `ProcessInfo.processInfo.arguments.contains("-ui-testing")` to use mock services and in-memory storage, so UI tests are fast and repeatable. Add `.accessibilityIdentifier("recipeGrid")` to views that have no text to find them.

:::tip Record UI tests
Put the cursor inside a UI test function and press the red **Record** button at the bottom of the editor. Xcode writes code as you tap around the app. It's great for learning the API.
:::

## Running tests

- [[Cmd]] + [[U]] runs all tests; the **Test navigator** ([[Cmd]] + [[6]]) shows results.
- Click the diamond next to a test or suite to run just that one.
- Enable **code coverage** in the scheme's Test settings to see which lines your tests exercise.
- From the terminal: `xcodebuild test -scheme RecipeBox -destination 'platform=iOS Simulator,name=iPhone 17'`.

## What to test first

1. **View models**: state after loading, errors, user actions. This gives the most value.
2. **Decoding and formatting**: JSON parsing, dates, prices.
3. **Services with logic**: caching rules, validation.
4. **A few UI tests** for the most important user journeys.

## Recap

:::recap
- Swift Testing: `@Test` functions, `#expect` checks, `try #require` to unwrap, `@Suite` to group, `arguments:` for parameterized tests.
- Depend on protocols so tests can inject mocks; `await` async code directly in tests.
- XCUITest drives the real app via accessibility; use launch arguments for mock data.
- [[Cmd]] + [[U]] runs everything.
:::

:::exercise
1. Write parameterized tests for a `TipCalculator` model (Lesson 21's `TipViewModel`): 15% of 100 is 15; rounding up 12.3 gives 13; 0 amount gives 0 total.
2. Write a test that the `OfflineQuotesViewModel` from Lesson 26 keeps its cached quotes and sets an error message when refresh fails. (Hint: a mock `QuoteAPI` that throws, and a `DiskCache` with a unique name per test.)
:::

:::solution Show solution for task 1
```swift nocheck
import Testing
@testable import TipApp

struct TipViewModelTests {
    @Test(arguments: [(100.0, 15, false, 15.0), (82.0, 15, true, 13.0), (0.0, 20, false, 0.0)])
    func tip(amount: Double, percent: Int, roundUp: Bool, expectedTip: Double) {
        let vm = TipViewModel()
        vm.amount = amount
        vm.percent = percent
        vm.roundUp = roundUp
        #expect(vm.tip == expectedTip)
        #expect(vm.total == amount + expectedTip)
    }
}
```
(82 × 15% = 12.3, which rounds up to 13.)
:::

## Check your understanding

```quiz
Q: What's the difference between `#expect` and `#require`?
- [ ] They're identical
- [x] #expect records a failure and continues; #require throws and stops the test
- [ ] #require is for UI tests only
- [ ] #expect only works with Booleans
> Use #require for preconditions like unwrapping an optional.

Q: Why does a mock service make view model tests better?
- [ ] It makes the app faster
- [x] Tests become fast, offline and deterministic, and can simulate errors on demand
- [ ] It's required by Swift Testing
- [ ] It tests the real server
> You control exactly what the dependency returns.

Q: How do UI tests find elements on screen?
- [ ] By pixel coordinates
- [x] Through accessibility: labels, identifiers and element types
- [ ] By Swift variable names
- [ ] By reading the source code
> XCUITest uses the same accessibility tree VoiceOver uses.

Q: What does `@testable import RecipeBox` do?
- [ ] Imports a testing library
- [x] Lets the test target access the app's internal types and functions
- [ ] Runs the app
- [ ] Marks the module as testable in the App Store
> Without it, tests could only see public declarations.
```
