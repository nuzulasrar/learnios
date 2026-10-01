---
id: capstone-finish
title: "Capstone 4: Test, Polish, Ship & What's Next"
part: 8
minutes: 45
summary: Finish RecipeBox like a professional. Write tests for decoding, the cache and the view model, add a UI test, polish accessibility and details, ship to TestFlight, then plan your next steps.
---

:::goals
- Unit tests for decoding, the cache and `HomeViewModel`, with Swift Testing
- A UI test with XCTest
- A polish checklist: icon, accessibility, localisation, edge cases
- Archiving and sending RecipeBox to TestFlight
- Where to go next
:::

## 1. Decoding tests

Open the `RecipeBoxTests` target and replace the template test file:

```swift title="RecipeBoxTests/RecipeDecodingTests.swift"
import Testing
import Foundation
@testable import RecipeBox

struct RecipeDecodingTests {
    let json = """
    {
      "recipes": [{
        "id": 7, "name": "Tacos", "ingredients": ["Tortillas", "Beef"],
        "instructions": ["Warm the tortillas.", "Fill and serve."],
        "prepTimeMinutes": 15, "cookTimeMinutes": 10, "servings": 3,
        "difficulty": "Easy", "cuisine": "Mexican", "caloriesPerServing": 450,
        "tags": ["Mexican"], "userId": 1, "image": "https://cdn.dummyjson.com/recipe-images/7.webp",
        "rating": 4.8, "reviewCount": 12, "mealType": ["Dinner"]
      }],
      "total": 1, "skip": 0, "limit": 1
    }
    """

    @Test func decodesTheAPIResponse() throws {
        let response = try JSONDecoder().decode(RecipeListResponse.self, from: Data(json.utf8))
        let recipe = try #require(response.recipes.first)

        #expect(recipe.name == "Tacos")
        #expect(recipe.totalMinutes == 25)
        #expect(recipe.imageURL?.lastPathComponent == "7.webp")
        #expect(recipe.ingredients.count == 2)
    }

    @Test func shareTextListsIngredients() {
        #expect(Recipe.sample.shareText.contains("• Pizza dough"))
    }
}
```

## 2. Cache tests

```swift title="RecipeBoxTests/RecipeCacheTests.swift"
import Testing
import Foundation
@testable import RecipeBox

struct RecipeCacheTests {
    let cache = RecipeCache(fileName: "test-\(UUID().uuidString).json")   // unique per test

    @Test func emptyCacheReturnsNil() async {
        #expect(await cache.load() == nil)
    }

    @Test func savesAndLoadsRecipes() async throws {
        try await cache.save(Recipe.samples)

        let snapshot = try #require(await cache.load())
        #expect(snapshot.recipes == Recipe.samples)
        await cache.clear()
    }
}
```

## 3. View model tests

This is where dependency injection pays off. Every scenario is a few lines, with no network:

```swift title="RecipeBoxTests/HomeViewModelTests.swift"
import Testing
import Foundation
@testable import RecipeBox

@MainActor
struct HomeViewModelTests {
    private func makeCache() -> RecipeCache {
        RecipeCache(fileName: "vm-test-\(UUID().uuidString).json")
    }

    @Test func loadsRecipesSortedByRating() async {
        let vm = HomeViewModel(service: MockRecipeService(), cache: makeCache())

        await vm.loadIfNeeded()

        #expect(vm.recipes.map(\.rating) == Recipe.samples.map(\.rating).sorted(by: >))
        #expect(vm.errorMessage == nil)
        #expect(vm.isLoading == false)
    }

    @Test func offlineWithoutCacheShowsError() async {
        let vm = HomeViewModel(service: MockRecipeService(error: URLError(.notConnectedToInternet)),
                               cache: makeCache())

        await vm.loadIfNeeded()

        #expect(vm.recipes.isEmpty)
        #expect(vm.errorMessage?.contains("Couldn't load") == true)
    }

    @Test func offlineWithCacheKeepsRecipes() async throws {
        let cache = makeCache()
        try await cache.save(Recipe.samples)
        let vm = HomeViewModel(service: MockRecipeService(error: URLError(.notConnectedToInternet)), cache: cache)

        await vm.loadIfNeeded()

        #expect(vm.recipes.count == Recipe.samples.count)
        #expect(vm.errorMessage?.contains("offline") == true)
    }

    @Test(arguments: [("pizza", 1), ("ITALIAN", 1), ("dessert", 1), ("", 3), ("sushi", 0)])
    func searchMatchesNameCuisineAndTags(query: String, expectedCount: Int) async {
        let vm = HomeViewModel(service: MockRecipeService(), cache: makeCache())
        await vm.loadIfNeeded()

        vm.searchText = query

        #expect(vm.visibleRecipes.count == expectedCount)
    }
}
```

Press [[Cmd]] + [[U]]. All green? 🎉 If a test fails, the message shows the actual values, so read it carefully before changing code.

## 4. A UI test

The app already swaps in mock data when launched with `-ui-testing` (see `RecipeBoxApp`). In the `RecipeBoxUITests` target:

```swift title="RecipeBoxUITests/RecipeBoxUITests.swift" nocheck
import XCTest

final class RecipeBoxUITests: XCTestCase {
    @MainActor
    func testFavouritingARecipeShowsItInFavourites() throws {
        let app = XCUIApplication()
        app.launchArguments = ["-ui-testing"]
        app.launch()

        let heart = app.buttons["Add Classic Margherita Pizza to favourites"]
        XCTAssertTrue(heart.waitForExistence(timeout: 5))
        heart.tap()

        app.tabBars.buttons["Favourites"].tap()
        XCTAssertTrue(app.staticTexts["Classic Margherita Pizza"].waitForExistence(timeout: 2))
    }
}
```

The test finds the heart by its **accessibility label**, another reason good labels matter.

## 5. Polish checklist

- [ ] **App icon**: a 1024×1024 image in AppIcon, with Dark and Tinted variants.
- [ ] **Launch screen**: in the target's **Info** tab, set a *Launch Screen* background colour that matches your app.
- [ ] **Accessibility**: navigate the whole app with **VoiceOver**; run **Xcode → Open Developer Tool → Accessibility Inspector → Audit**; try the largest Dynamic Type size and **Bold Text**.
- [ ] **Localisation**: add a String Catalog, build to collect strings, and add a second language.
- [ ] **Dark mode** on every screen.
- [ ] **Edge cases**: first launch offline, empty search results, favourites with no internet, rotating on every screen, being terminated in the background.
- [ ] **iPad**: check the grid and navigation on an iPad simulator, or remove iPad from *Supported Destinations* if you don't want to support it yet.
- [ ] **Performance**: scroll the grid fast on a real device; profile with Instruments if anything stutters.
- [ ] **Haptics and animation**: subtle `sensoryFeedback` on important actions, short animations, respect Reduce Motion.

## 6. Ship it to TestFlight

Follow Lesson 33:

1. Set **Version** `1.0.0` and **Build** `1`.
2. Select **Any iOS Device (arm64)**, then **Product → Archive**.
3. **Distribute App → App Store Connect → Upload**.
4. In App Store Connect, add yourself (and friends) as testers, then install from the **TestFlight** app. Now you're running your own app from the App Store's infrastructure! 🚀
5. When you're ready: screenshots, description, privacy details ("Data Not Collected" if you added no analytics), a privacy policy URL, and **Submit for Review**.

:::tip Put it on GitHub
Push RecipeBox to a public GitHub repository with a README showing screenshots, the feature list, the architecture diagram and the tech stack. It's an excellent portfolio piece for iOS job applications.
:::

## 🎓 Congratulations!

You started with "What is iOS?" and you've built a real, offline-first app with:

<div class="diagram"><div class="flow">
<div class="hl">Swift<small>types · optionals · async/await · actors</small></div>
<div class="hl">SwiftUI<small>layout · state · lists · navigation · animation</small></div>
<div class="hl">Architecture<small>MVVM · Observation · DI</small></div>
<div class="hl">Data<small>URLSession · Codable · SwiftData · caching</small></div>
<div class="hl">Quality<small>Swift Testing · debugging · TestFlight</small></div>
</div></div>

That's the modern Apple toolkit professionals use every day. Seriously, well done. 🎉

## Where to go next

### Build more apps

The best way to grow is to build things *without* a tutorial. Ideas, roughly in order of difficulty:

1. **Habit tracker**: SwiftData, local notifications, charts with **Swift Charts**, a widget.
2. **Expense tracker**: forms, validation, SwiftData relationships, CSV export with `ShareLink`, Face ID lock with **LocalAuthentication**.
3. **Weather app**: location, a real API with a key (kept out of Git), WidgetKit, maps with **MapKit**.
4. **Chat app**: Sign in with Apple, a backend (e.g. CloudKit or Firebase), push notifications, photo messages.

### Topics to learn next

| Topic | Why |
|---|---|
| **Swift Charts** | Beautiful, accessible charts in a few lines |
| **MapKit & Core Location** | Maps, search and directions |
| **CloudKit / SwiftData sync** | Sync data across the user's devices via iCloud |
| **StoreKit 2** | In-app purchases and subscriptions |
| **Live Activities & Dynamic Island** | Real-time updates on the Lock Screen |
| **Apple Intelligence & App Intents** | Integrate your app with Siri and system intelligence features |
| **Foundation Models / Core ML** | On-device machine learning and language features |
| **visionOS, watchOS, macOS** | Bring your SwiftUI skills to every Apple platform |
| **Swift packages & modularisation** | How larger teams structure apps |
| **CI/CD** with Xcode Cloud or GitHub Actions | Automated builds, tests and TestFlight uploads |

### Resources

- [developer.apple.com/tutorials](https://developer.apple.com/tutorials/app-dev-training): Apple's free, hands-on SwiftUI tutorials.
- [The Swift Programming Language](https://docs.swift.org/swift-book/): the official book, free online.
- [Apple Developer Documentation](https://developer.apple.com/documentation/) and **WWDC session videos** (in the Apple Developer app): the best way to learn each year's new APIs.
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/): how great iOS apps look and behave.
- [Swift Forums](https://forums.swift.org) and the Apple Developer Forums: ask questions and learn from others.

:::analogy One last piece of advice
Every experienced iOS developer still searches the docs, watches WWDC videos to catch up, and gets confused by signing errors. That's normal! What makes a professional isn't knowing everything but knowing how to find out. You now have the foundation to learn anything else on Apple platforms. Go build something you care about. 🚀
:::

## Final check

```quiz
Q: Why do the view model tests create a RecipeCache with a unique file name?
- [ ] File names must be unique on iOS
- [x] So tests don't share cached data with each other or with the real app
- [ ] It makes tests faster
- [ ] SwiftData requires it
> Isolated tests are reliable tests.

Q: How does the UI test get predictable data without the network?
- [ ] It turns off Wi-Fi
- [x] It launches the app with "-ui-testing", which makes the app use MockRecipeService and an in-memory database
- [ ] It edits the API
- [ ] UI tests can't use mock data
> Launch arguments let the app switch dependencies for testing.

Q: What's the best way to check your app's accessibility?
- [ ] Only look at it in dark mode
- [x] Navigate it with VoiceOver, run the Accessibility Inspector audit, and try large text sizes
- [ ] Add accessibility labels to every Text
- [ ] Run the unit tests
> Experiencing the app like assistive-technology users do reveals real problems quickly.
```
