---
id: architecture-di
title: App Architecture & Dependency Injection
part: 4
minutes: 30
summary: Structure an app so it scales. Covers layers, services behind protocols, dependency injection through initialisers and the environment, previews with mock data, and a folder structure that grows with you.
---

:::goals
- The layers of a well-structured app
- Services (repositories) behind protocols
- **Dependency injection** through initialisers
- Mocks for previews and tests
- Injecting dependencies through the SwiftUI environment
- A folder structure that scales, and Swift packages for modules
:::

## Why architecture?

A small app can live in a few files. As it grows, code without structure becomes a tangle: views call the network directly, the same data is loaded in five places, and changing one thing breaks another. **Architecture** means rules for where code goes, so that:

- each type has **one clear job**,
- parts can be **tested** and **previewed** in isolation,
- you can **swap** one part (e.g. the API) without rewriting the rest.

## The layers

<div class="diagram">
<div class="layers">
<div class="layer ui"><h5>Presentation</h5><div class="boxes"><span>SwiftUI Views</span><span>@Observable ViewModels</span></div></div>
<div class="dir">↓ calls async methods · ↑ returns models</div>
<div class="layer domain"><h5>Services / Repositories</h5><div class="boxes"><span>RecipeService (protocol)</span><span>LiveRecipeService</span><span>MockRecipeService</span></div></div>
<div class="dir">↓ ↑</div>
<div class="layer data"><h5>Data sources</h5><div class="boxes"><span>URLSession (network)</span><span>SwiftData (database)</span><span>UserDefaults / Keychain / files</span></div></div>
</div>
<div class="diagram-caption">Dependencies point downwards. Lower layers never know about views.</div>
</div>

- **Views** display state and forward user actions.
- **View models** hold screen state and coordinate: "load recipes, filter them, handle errors".
- **Services** (also called repositories or clients) know *how* to get and save data: which URL, which database query, which cache.

## Services behind protocols

Define **what** the app needs as a protocol:

```swift title="RecipeService.swift"
import Foundation

struct Recipe: Identifiable, Hashable, Codable, Sendable {
    let id: Int
    let name: String
    let cuisine: String
}

protocol RecipeService: Sendable {
    func fetchRecipes() async throws -> [Recipe]
}
```

Then write implementations. The real one talks to the network (Lesson 23 builds it properly):

```swift title="LiveRecipeService.swift"
import Foundation

struct LiveRecipeService: RecipeService {
    func fetchRecipes() async throws -> [Recipe] {
        let url = URL(string: "https://dummyjson.com/recipes?limit=0&select=name,cuisine")!
        let (data, _) = try await URLSession.shared.data(from: url)
        struct Response: Decodable { let recipes: [Recipe] }
        return try JSONDecoder().decode(Response.self, from: data).recipes
    }
}
```

And a mock returns sample data instantly, which is perfect for previews and tests:

```swift title="MockRecipeService.swift"
import Foundation

struct MockRecipeService: RecipeService {
    var recipes: [Recipe] = [
        Recipe(id: 1, name: "Margherita Pizza", cuisine: "Italian"),
        Recipe(id: 2, name: "Pad Thai", cuisine: "Thai")
    ]
    var shouldFail = false

    func fetchRecipes() async throws -> [Recipe] {
        if shouldFail { throw URLError(.notConnectedToInternet) }
        return recipes
    }
}
```

## Dependency injection

**Dependency injection (DI)** means a type **receives** the things it depends on instead of creating them itself. The simplest form is passing them into the initialiser:

```swift title="RecipeListViewModel.swift"
import SwiftUI

@MainActor
@Observable
final class RecipeListViewModel {
    private let service: any RecipeService          // depends on the PROTOCOL
    private(set) var recipes: [Recipe] = []
    private(set) var errorMessage: String?

    init(service: any RecipeService) {             // injected
        self.service = service
    }

    func load() async {
        do {
            recipes = try await service.fetchRecipes()
            errorMessage = nil
        } catch {
            errorMessage = "Couldn't load recipes."
        }
    }
}

struct RecipeListScreen: View {
    @State private var viewModel: RecipeListViewModel

    init(service: any RecipeService) {
        _viewModel = State(initialValue: RecipeListViewModel(service: service))
    }

    var body: some View {
        List(viewModel.recipes) { recipe in
            LabeledContent(recipe.name, value: recipe.cuisine)
        }
        .overlay {
            if let message = viewModel.errorMessage {
                ContentUnavailableView(message, systemImage: "wifi.slash")
            }
        }
        .task { await viewModel.load() }
    }
}

#Preview("Sample data") {
    RecipeListScreen(service: MockRecipeService())
}

#Preview("Error") {
    RecipeListScreen(service: MockRecipeService(shouldFail: true))
}
```

Now:

- The **app** passes `LiveRecipeService()`.
- **Previews** pass `MockRecipeService()`: instant, offline, and able to show the error state on demand.
- **Tests** pass a mock and check the view model's behaviour (Lesson 31).

`_viewModel = State(initialValue: …)` is how you initialise a `@State` property from an initialiser parameter. The underscore accesses the property wrapper itself.

:::analogy
Without DI, a class goes shopping for its own ingredients, so you can't give it different ones. With DI, ingredients are delivered to the kitchen door. In tests you deliver fake ingredients, and the cook never knows the difference.
:::

## Injecting through the environment

Passing services through every initialiser gets tedious when they're needed deep in the view tree. SwiftUI's environment can carry them. Define an environment value with the `@Entry` macro:

```swift title="Dependencies.swift"
import SwiftUI

extension EnvironmentValues {
    @Entry var recipeService: any RecipeService = LiveRecipeService()   // default value
}
```

Read it in any view, and override it wherever you like:

```swift
struct RecipesRoot: View {
    @Environment(\.recipeService) private var service

    var body: some View {
        RecipeListScreen(service: service)
    }
}

#Preview {
    RecipesRoot()
        .environment(\.recipeService, MockRecipeService())   // swap in the mock
}
```

Choose whichever is clearer: initialiser injection is explicit and great for view models; the environment is convenient for app-wide services.

## A folder structure that scales

Group by **feature**, with shared code alongside:

<div class="tree">RecipeBox/
├── App/
│   ├── RecipeBoxApp.swift
│   └── Dependencies.swift
├── Models/                 ← Recipe, Ingredient…
├── Services/               ← RecipeService, LiveRecipeService, RecipeCache
├── Features/
│   ├── Home/               ← HomeView.swift, HomeViewModel.swift
│   ├── Detail/
│   ├── Favorites/
│   └── Settings/
├── Components/             ← reusable views: RecipeCard, RatingView…
├── Resources/              ← Assets.xcassets, Localizable.xcstrings
└── Preview Content/        ← mocks and sample data</div>

For large apps, teams split features into **local Swift packages** (**File → New → Package**) so each module builds and tests independently. For this course, folders are plenty.

:::note Other architectures
You'll hear about alternatives like **TCA** (The Composable Architecture), **VIPER** and **Clean Architecture**. They share the same goals: separation, testability, clear data flow. MVVM with injected services is the most common starting point, and it's what the capstone uses.
:::

## Recap

:::recap
- Views → view models → services → data sources. Dependencies point down.
- Hide data access behind **protocols**; write live and mock implementations.
- **Inject** dependencies through initialisers (or the environment with `@Entry`).
- Mocks make previews instant and tests reliable.
- Organise folders by feature.
:::

:::exercise
1. Sketch the layers for a **weather app** that shows today's forecast (from an API) and a list of saved cities (stored locally). Name the view models, the service protocols and their methods, and the data sources.
2. Take `QuotesViewModel` from the previous lesson and extract the fake network call into a `QuoteService` protocol with a mock that can be told to fail. Inject it, and add two previews: success and failure.
:::

:::solution Show a possible answer for task 1
- **Views / view models:** `ForecastView` + `ForecastViewModel`; `CitiesView` + `CitiesViewModel`.
- **Services:** `protocol WeatherService { func forecast(for city: City) async throws -> Forecast }` and `protocol CityStore { func savedCities() throws -> [City]; func save(_ city: City) throws; func delete(_ city: City) throws }`.
- **Implementations:** `LiveWeatherService` (URLSession), `SwiftDataCityStore` (SwiftData); `MockWeatherService` and `InMemoryCityStore` for previews and tests.
- View models receive `any WeatherService` and `any CityStore` in their initialisers.
:::

## Check your understanding

```quiz
Q: What is dependency injection?
- [ ] Downloading packages
- [x] Giving a type its dependencies from outside instead of creating them itself
- [ ] Injecting code at runtime
- [ ] A way to speed up builds
> DI decouples a type from the concrete implementations it uses.

Q: Why should a view model depend on `any RecipeService` rather than `LiveRecipeService`?
- [ ] Protocols are faster
- [x] So previews and tests can pass a mock implementation
- [ ] Swift requires protocols
- [ ] Live services can't be stored
> Depending on the protocol makes the implementation swappable.

Q: Which layer should contain URLSession code?
- [ ] Views
- [ ] View models
- [x] Services / data sources
- [ ] The App struct
> Views and view models shouldn't know how data is fetched.

Q: How do you initialise a `@State` property from an init parameter?
- [x] `_viewModel = State(initialValue: MyViewModel(service: service))`
- [ ] `self.viewModel = service`
- [ ] `@State(service) var viewModel`
- [ ] You can't
> The underscore accesses the State property wrapper directly.
```
