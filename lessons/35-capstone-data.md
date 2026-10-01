---
id: capstone-data
title: "Capstone 2: Services, Cache & Favourites"
part: 8
minutes: 45
summary: Build RecipeBox's data layer. A RecipeService protocol with live and mock implementations, an offline disk cache, a SwiftData favourites model, and the HomeViewModel that ties them together.
---

:::goals
- A `RecipeService` protocol with a live (network) and a mock implementation
- A `RecipeCache` actor that saves the last response to disk
- The `FavoriteRecipe` SwiftData model, storing the full recipe for offline use
- `HomeViewModel`: show cached data instantly, refresh from the network, search
:::

## The data flow

<div class="diagram"><div class="flow">
<div>DummyJSON API</div><div class="arrow">→</div>
<div class="hl">LiveRecipeService</div><div class="arrow">→</div>
<div>HomeViewModel</div><div class="arrow">⇄</div>
<div class="hl">RecipeCache<small>disk</small></div>
</div>
<div class="flow" style="margin-top:8px">
<div>Views</div><div class="arrow">⇄</div><div class="hl">SwiftData<small>FavoriteRecipe</small></div>
</div></div>

## 1. The service

```swift title="Services/RecipeService.swift"
import Foundation

protocol RecipeService: Sendable {
    func fetchRecipes() async throws -> [Recipe]
}

enum RecipeServiceError: LocalizedError {
    case badStatus(Int)

    var errorDescription: String? {
        switch self {
        case .badStatus(let code): "The recipe server had a problem (\(code)). Please try again later."
        }
    }
}

struct LiveRecipeService: RecipeService {
    var session: URLSession = .shared
    private let url = URL(string: "https://dummyjson.com/recipes?limit=0")!   // limit=0 → all recipes

    func fetchRecipes() async throws -> [Recipe] {
        let (data, response) = try await session.data(from: url)
        if let http = response as? HTTPURLResponse, !(200..<300).contains(http.statusCode) {
            throw RecipeServiceError.badStatus(http.statusCode)
        }
        return try JSONDecoder().decode(RecipeListResponse.self, from: data).recipes
    }
}

struct MockRecipeService: RecipeService {
    var recipes: [Recipe] = Recipe.samples
    var error: (any Error)? = nil
    var delay: Duration = .zero

    func fetchRecipes() async throws -> [Recipe] {
        if delay > .zero { try await Task.sleep(for: delay) }
        if let error { throw error }
        return recipes
    }
}
```

The mock can simulate **success**, **failure** (`error: URLError(.notConnectedToInternet)`) and **slowness** (`delay: .seconds(2)`), which is perfect for previews of every state.

## 2. The disk cache

```swift title="Services/RecipeCache.swift"
import Foundation

/// Saves the most recent list of recipes to the Caches folder so the app works offline.
actor RecipeCache {
    struct Snapshot: Codable, Sendable {
        let savedAt: Date
        let recipes: [Recipe]
    }

    private let fileURL: URL

    init(fileName: String = "recipes-cache.json") {
        fileURL = URL.cachesDirectory.appending(path: fileName)
    }

    func load() -> Snapshot? {
        guard let data = try? Data(contentsOf: fileURL) else { return nil }
        return try? JSONDecoder().decode(Snapshot.self, from: data)
    }

    func save(_ recipes: [Recipe]) throws {
        let data = try JSONEncoder().encode(Snapshot(savedAt: .now, recipes: recipes))
        try data.write(to: fileURL, options: .atomic)
    }

    func clear() {
        try? FileManager.default.removeItem(at: fileURL)
    }
}
```

Tests can pass a unique `fileName` so they never interfere with each other or with the real app's cache.

## 3. The favourites model (SwiftData)

Favourites must work **offline** and survive even if the API changes, so we store the whole recipe (encoded as JSON) alongside a few fields we want to sort and display:

```swift title="Models/FavoriteRecipe.swift"
import Foundation
import SwiftData

@Model
final class FavoriteRecipe {
    @Attribute(.unique) var recipeID: Int
    var name: String
    var cuisine: String
    var totalMinutes: Int
    var imageURLString: String
    var addedAt: Date
    var payload: Data                       // the full Recipe, JSON-encoded

    init(recipe: Recipe) {
        recipeID = recipe.id
        name = recipe.name
        cuisine = recipe.cuisine
        totalMinutes = recipe.totalMinutes
        imageURLString = recipe.image
        addedAt = .now
        payload = (try? JSONEncoder().encode(recipe)) ?? Data()
    }

    /// The full recipe, decoded from the stored JSON.
    var recipe: Recipe? {
        try? JSONDecoder().decode(Recipe.self, from: payload)
    }
}
```

And a small helper that adds or removes a favourite, given the current favourites from a `@Query`:

```swift title="Models/FavoriteRecipe+Toggle.swift"
import SwiftData

extension ModelContext {
    /// Adds the recipe to favourites, or removes it if it's already there.
    func toggleFavorite(_ recipe: Recipe, in favorites: [FavoriteRecipe]) {
        if let existing = favorites.first(where: { $0.recipeID == recipe.id }) {
            delete(existing)
        } else {
            insert(FavoriteRecipe(recipe: recipe))
        }
    }
}
```

`@Attribute(.unique)` guarantees one row per recipe, even if something inserts twice.

## 4. The home view model

This is the brain of the Home screen. It follows the offline-first pattern from Lesson 26:

```swift title="Features/Home/HomeViewModel.swift"
import Foundation
import Observation

@MainActor
@Observable
final class HomeViewModel {
    private(set) var recipes: [Recipe] = []
    private(set) var isLoading = false
    private(set) var errorMessage: String?
    private(set) var lastUpdated: Date?
    var searchText = ""

    private let service: any RecipeService
    private let cache: RecipeCache
    private var hasLoaded = false

    init(service: any RecipeService, cache: RecipeCache = RecipeCache()) {
        self.service = service
        self.cache = cache
    }

    /// Recipes matching the search text (name, cuisine or tag).
    var visibleRecipes: [Recipe] {
        let query = searchText.trimmingCharacters(in: .whitespaces)
        guard !query.isEmpty else { return recipes }
        return recipes.filter { recipe in
            recipe.name.localizedCaseInsensitiveContains(query)
                || recipe.cuisine.localizedCaseInsensitiveContains(query)
                || recipe.tags.contains { $0.localizedCaseInsensitiveContains(query) }
        }
    }

    /// Called when the screen appears: show the cache instantly, then refresh once.
    func loadIfNeeded() async {
        guard !hasLoaded else { return }
        hasLoaded = true
        if let snapshot = await cache.load() {
            recipes = snapshot.recipes
            lastUpdated = snapshot.savedAt
        }
        await refresh()
    }

    /// Fetches fresh recipes from the network (pull-to-refresh uses this too).
    func refresh() async {
        isLoading = true
        defer { isLoading = false }
        do {
            let fresh = try await service.fetchRecipes()
            recipes = fresh.sorted { $0.rating > $1.rating }
            lastUpdated = .now
            errorMessage = nil
            try? await cache.save(recipes)
        } catch is CancellationError {
            // The view disappeared: nothing to report.
        } catch {
            errorMessage = recipes.isEmpty
                ? "Couldn't load recipes. Check your internet connection and try again."
                : "You're offline. Showing saved recipes."
        }
    }
}
```

Walk through what happens on first launch **with** internet, first launch **without** internet, and a later launch **without** internet:

| Situation | What the user sees |
|---|---|
| First launch, online | Spinner → recipes (and they're cached) |
| First launch, offline | Spinner → "Couldn't load recipes…" with a Try Again button |
| Later launch, offline | Cached recipes **instantly**, plus a small "You're offline" note |

## 5. Check it compiles

Build with [[Cmd]] + [[B]]. There's nothing new to see on screen yet, but a clean build means your data layer is ready. Commit: *"Data layer: service, cache, favourites, view model"*.

:::exercise Stretch goals
1. Add a `mealType` filter to `HomeViewModel`: a `selectedMealType: String?` property and a computed `allMealTypes: [String]` (unique, sorted), applied in `visibleRecipes`.
2. Make `loadIfNeeded()` skip the network refresh if the cache is less than 15 minutes old. (Keep pull-to-refresh always refreshing.)
:::

## Check your understanding

```quiz
Q: Why store the full recipe JSON in FavoriteRecipe instead of only its id?
- [ ] SwiftData can't store Ints
- [x] So favourites can be displayed and opened offline, without re-fetching from the API
- [ ] To make the database bigger
- [ ] It's required by @Model
> Keeping a local copy makes favourites fully offline.

Q: Why is RecipeCache an actor?
- [ ] Actors are required for file access
- [x] It protects the cache file from simultaneous reads and writes by different tasks
- [ ] To run on the main thread
- [ ] For SwiftData
> Actors serialise access to shared resources.

Q: What does `loadIfNeeded()` do before calling the network?
- [ ] Nothing
- [x] Shows the cached snapshot, if there is one, so the screen isn't empty
- [ ] Deletes the cache
- [ ] Checks the network with NWPathMonitor
> Show cached data instantly, then refresh.

Q: A refresh fails but cached recipes are showing. What's the error message?
- [ ] "Couldn't load recipes…"
- [x] "You're offline. Showing saved recipes."
- [ ] No message at all
- [ ] The raw URLError
> The message depends on whether the user still has something to look at.
```
