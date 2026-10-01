---
id: capstone-ui
title: "Capstone 3: Screens, Navigation & Settings"
part: 8
minutes: 60
summary: Bring RecipeBox to life. Build reusable components, the Home grid with search and refresh, a rich detail screen, the favourites list, settings, and the tab-based app shell.
---

:::goals
- Reusable components: `RecipeImage`, `RecipeCard`, `FavoriteToggle`
- The Home screen: adaptive grid, search, pull-to-refresh, empty and error states
- The detail screen: header image, facts, ingredients, steps, share and favourite
- The favourites list with SwiftData `@Query` and swipe to delete
- Settings with `@AppStorage`, and the `TabView` app shell
:::

## 1. Components

### `RecipeImage`

`AsyncImage` with a nice placeholder and failure state, used everywhere a recipe photo appears:

```swift title="Components/RecipeImage.swift"
import SwiftUI

struct RecipeImage: View {
    let url: URL?

    var body: some View {
        AsyncImage(url: url, transaction: Transaction(animation: .easeInOut)) { phase in
            switch phase {
            case .success(let image):
                image.resizable().scaledToFill()
            case .failure:
                placeholder(systemImage: "fork.knife")
            default:
                placeholder(systemImage: nil)
            }
        }
        .accessibilityHidden(true)            // decorative: the recipe name is always shown nearby
    }

    private func placeholder(systemImage: String?) -> some View {
        Rectangle()
            .fill(.orange.opacity(0.12))
            .overlay {
                if let systemImage {
                    Image(systemName: systemImage).font(.largeTitle).foregroundStyle(.orange)
                } else {
                    ProgressView()
                }
            }
    }
}
```

### `FavoriteToggle`

A heart button that reads favourites with `@Query` and toggles them in SwiftData. Because it queries by itself, any screen can drop it in:

```swift title="Components/FavoriteToggle.swift"
import SwiftUI
import SwiftData

struct FavoriteToggle: View {
    let recipe: Recipe
    @Environment(\.modelContext) private var context
    @Query private var favorites: [FavoriteRecipe]

    private var isFavorite: Bool { favorites.contains { $0.recipeID == recipe.id } }

    var body: some View {
        Button {
            withAnimation(.snappy) {
                context.toggleFavorite(recipe, in: favorites)
            }
        } label: {
            Image(systemName: isFavorite ? "heart.fill" : "heart")
                .foregroundStyle(isFavorite ? .red : .primary)
                .contentTransition(.symbolEffect(.replace))
        }
        .sensoryFeedback(.selection, trigger: isFavorite)       // a light haptic tap
        .accessibilityLabel(isFavorite ? "Remove \(recipe.name) from favourites"
                                       : "Add \(recipe.name) to favourites")
    }
}
```

### `RecipeCard`

```swift title="Components/RecipeCard.swift"
import SwiftUI

struct RecipeCard: View {
    let recipe: Recipe

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            RecipeImage(url: recipe.imageURL)
                .frame(height: 180)
                .frame(maxWidth: .infinity)
                .clipped()

            VStack(alignment: .leading, spacing: 6) {
                Text(recipe.name)
                    .font(.headline)
                    .foregroundStyle(.primary)
                    .lineLimit(1)
                HStack(spacing: 14) {
                    Label("\(recipe.totalMinutes) min", systemImage: "clock")
                    Label(recipe.rating.formatted(.number.precision(.fractionLength(1))), systemImage: "star.fill")
                    Label(recipe.cuisine, systemImage: "globe")
                        .lineLimit(1)
                }
                .font(.caption)
                .foregroundStyle(.secondary)
                .labelStyle(.titleAndIcon)
            }
            .padding(14)
        }
        .background(.background.secondary)
        .clipShape(.rect(cornerRadius: 20))
        .contentShape(.rect(cornerRadius: 20))
        .accessibilityElement(children: .combine)
    }
}
```

`.accessibilityElement(children: .combine)` makes VoiceOver read the whole card as one item: *"Classic Margherita Pizza, 35 min, 4.6, Italian"*.

Notice the card has **no heart button inside it**. The whole card will be wrapped in a `NavigationLink`, and a button nested inside another button is unreliable for taps, VoiceOver and UI tests. Instead, Home overlays the heart *on top of* the link.

## 2. Home

```swift title="Features/Home/HomeView.swift"
import SwiftUI

struct HomeView: View {
    @State private var viewModel: HomeViewModel

    init(viewModel: HomeViewModel) {
        _viewModel = State(initialValue: viewModel)
    }

    private let columns = [GridItem(.adaptive(minimum: 300), spacing: 16)]

    var body: some View {
        NavigationStack {
            ScrollView {
                if let message = viewModel.errorMessage, !viewModel.recipes.isEmpty {
                    Label(message, systemImage: "wifi.slash")
                        .font(.footnote)
                        .foregroundStyle(.secondary)
                        .padding(.horizontal)
                }
                LazyVGrid(columns: columns, spacing: 16) {
                    ForEach(viewModel.visibleRecipes) { recipe in
                        NavigationLink(value: recipe) {
                            RecipeCard(recipe: recipe)
                        }
                        .buttonStyle(.plain)
                        .overlay(alignment: .topTrailing) {       // a sibling of the link, not inside it
                            FavoriteToggle(recipe: recipe)
                                .font(.title3)
                                .padding(10)
                                .background(.thinMaterial, in: .circle)
                                .padding(10)
                        }
                    }
                }
                .padding()
            }
            .background(Color(.systemGroupedBackground))
            .overlay { stateOverlay }
            .navigationTitle("RecipeBox")
            .navigationDestination(for: Recipe.self) { recipe in
                RecipeDetailView(recipe: recipe)
            }
            .searchable(text: $viewModel.searchText, prompt: "Recipes, cuisines, tags")
            .refreshable { await viewModel.refresh() }
            .task { await viewModel.loadIfNeeded() }
        }
    }

    @ViewBuilder
    private var stateOverlay: some View {
        if viewModel.recipes.isEmpty {
            if viewModel.isLoading {
                ProgressView("Loading recipes…")
            } else if let message = viewModel.errorMessage {
                ContentUnavailableView {
                    Label("No Recipes", systemImage: "wifi.slash")
                } description: {
                    Text(message)
                } actions: {
                    Button("Try Again") { Task { await viewModel.refresh() } }
                        .buttonStyle(.borderedProminent)
                }
            }
        } else if viewModel.visibleRecipes.isEmpty {
            ContentUnavailableView.search(text: viewModel.searchText)
        }
    }
}

#Preview("Loaded") {
    HomeView(viewModel: HomeViewModel(service: MockRecipeService(), cache: RecipeCache(fileName: "preview.json")))
        .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}

#Preview("Offline, no cache") {
    HomeView(viewModel: HomeViewModel(service: MockRecipeService(error: URLError(.notConnectedToInternet)),
                                      cache: RecipeCache(fileName: "empty-preview.json")))
        .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}
```

Notice how the two previews show **different states** just by injecting a different mock. That's the payoff of dependency injection.

## 3. Recipe detail

```swift title="Features/Detail/RecipeDetailView.swift"
import SwiftUI

struct RecipeDetailView: View {
    let recipe: Recipe

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 24) {
                RecipeImage(url: recipe.imageURL)
                    .frame(height: 300)
                    .frame(maxWidth: .infinity)
                    .clipped()

                VStack(alignment: .leading, spacing: 8) {
                    Text(recipe.name)
                        .font(.largeTitle.bold())
                    Text("\(recipe.cuisine) · \(recipe.difficulty)")
                        .font(.title3)
                        .foregroundStyle(.secondary)
                }
                .padding(.horizontal)

                factsRow
                    .padding(.horizontal)

                section(title: "Ingredients", systemImage: "basket") {
                    ForEach(recipe.ingredients, id: \.self) { ingredient in
                        Label(ingredient, systemImage: "circle.fill")
                            .labelStyle(BulletLabelStyle())
                    }
                }

                section(title: "Steps", systemImage: "list.number") {
                    ForEach(Array(recipe.instructions.enumerated()), id: \.offset) { index, step in
                        HStack(alignment: .firstTextBaseline, spacing: 12) {
                            Text("\(index + 1)")
                                .font(.headline)
                                .foregroundStyle(.white)
                                .frame(width: 28, height: 28)
                                .background(.orange, in: .circle)
                            Text(step)
                                .fixedSize(horizontal: false, vertical: true)
                        }
                        .accessibilityElement(children: .combine)
                    }
                }
            }
            .padding(.bottom, 32)
        }
        .ignoresSafeArea(edges: .top)
        .navigationBarTitleDisplayMode(.inline)
        .toolbar {
            ToolbarItem(placement: .topBarTrailing) {
                ShareLink(item: recipe.shareText, subject: Text(recipe.name)) {
                    Label("Share", systemImage: "square.and.arrow.up")
                }
            }
            ToolbarItem(placement: .topBarTrailing) {
                FavoriteToggle(recipe: recipe)
            }
        }
    }

    private var factsRow: some View {
        HStack {
            fact(value: "\(recipe.totalMinutes)", label: "minutes", systemImage: "clock")
            Divider()
            fact(value: "\(recipe.servings)", label: "servings", systemImage: "person.2")
            Divider()
            fact(value: "\(recipe.caloriesPerServing)", label: "kcal", systemImage: "flame")
            Divider()
            fact(value: recipe.rating.formatted(.number.precision(.fractionLength(1))), label: "rating", systemImage: "star")
        }
        .frame(maxWidth: .infinity)
        .padding(.vertical, 12)
        .background(.background.secondary, in: .rect(cornerRadius: 16))
    }

    private func fact(value: String, label: String, systemImage: String) -> some View {
        VStack(spacing: 4) {
            Image(systemName: systemImage).foregroundStyle(.orange)
            Text(value).font(.headline)
            Text(label).font(.caption).foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .accessibilityElement(children: .combine)
    }

    private func section<Content: View>(title: String, systemImage: String,
                                        @ViewBuilder content: () -> Content) -> some View {
        VStack(alignment: .leading, spacing: 12) {
            Label(title, systemImage: systemImage)
                .font(.title2.bold())
            content()
        }
        .padding(.horizontal)
    }
}

struct BulletLabelStyle: LabelStyle {
    func makeBody(configuration: Configuration) -> some View {
        HStack(alignment: .firstTextBaseline, spacing: 10) {
            configuration.icon
                .font(.system(size: 6))
                .foregroundStyle(.orange)
            configuration.title
        }
    }
}

#Preview {
    NavigationStack {
        RecipeDetailView(recipe: .sample)
    }
    .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}
```

The `section(title:systemImage:content:)` helper takes a **`@ViewBuilder` closure**, the same mechanism `VStack` uses, so each section's content reads naturally.

## 4. Favourites

```swift title="Features/Favorites/FavoritesView.swift"
import SwiftUI
import SwiftData

struct FavoritesView: View {
    @Environment(\.modelContext) private var context
    @Query(sort: \FavoriteRecipe.addedAt, order: .reverse) private var favorites: [FavoriteRecipe]

    var body: some View {
        NavigationStack {
            List {
                ForEach(favorites) { favorite in
                    if let recipe = favorite.recipe {
                        NavigationLink(value: recipe) {
                            HStack(spacing: 12) {
                                RecipeImage(url: URL(string: favorite.imageURLString))
                                    .frame(width: 64, height: 64)
                                    .clipShape(.rect(cornerRadius: 12))
                                VStack(alignment: .leading, spacing: 4) {
                                    Text(favorite.name).font(.headline)
                                    Text("\(favorite.cuisine) · \(favorite.totalMinutes) min")
                                        .font(.subheadline)
                                        .foregroundStyle(.secondary)
                                }
                            }
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets { context.delete(favorites[index]) }
                }
            }
            .overlay {
                if favorites.isEmpty {
                    ContentUnavailableView("No Favourites Yet", systemImage: "heart",
                                           description: Text("Tap the heart on any recipe to save it here. Favourites work offline."))
                }
            }
            .navigationTitle("Favourites")
            .navigationDestination(for: Recipe.self) { recipe in
                RecipeDetailView(recipe: recipe)
            }
            .toolbar {
                if !favorites.isEmpty { EditButton() }
            }
        }
    }
}

#Preview {
    FavoritesView()
        .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}
```

`@Query` keeps this list in sync automatically: heart a recipe on the Home tab and it's already here when you switch tabs.

## 5. Settings

```swift title="Features/Settings/Appearance.swift"
import SwiftUI

enum Appearance: String, CaseIterable, Identifiable {
    case system, light, dark
    var id: Self { self }

    var title: String {
        switch self {
        case .system: "System"
        case .light: "Light"
        case .dark: "Dark"
        }
    }

    var colorScheme: ColorScheme? {
        switch self {
        case .system: nil
        case .light: .light
        case .dark: .dark
        }
    }
}
```

```swift title="Features/Settings/SettingsView.swift"
import SwiftUI
import SwiftData

struct SettingsView: View {
    @AppStorage("appearance") private var appearance = Appearance.system
    @Environment(\.modelContext) private var context
    @Query private var favorites: [FavoriteRecipe]
    @State private var confirmClear = false

    private var version: String {
        let info = Bundle.main.infoDictionary
        let short = info?["CFBundleShortVersionString"] as? String ?? "1.0"
        let build = info?["CFBundleVersion"] as? String ?? "1"
        return "\(short) (\(build))"
    }

    var body: some View {
        NavigationStack {
            Form {
                Section("Appearance") {
                    Picker("Theme", selection: $appearance) {
                        ForEach(Appearance.allCases) { Text($0.title).tag($0) }
                    }
                    .pickerStyle(.segmented)
                }

                Section("Favourites") {
                    LabeledContent("Saved recipes", value: "\(favorites.count)")
                    Button("Remove All Favourites", role: .destructive) { confirmClear = true }
                        .disabled(favorites.isEmpty)
                }

                Section("About") {
                    LabeledContent("Version", value: version)
                    Link("Recipes from DummyJSON", destination: URL(string: "https://dummyjson.com/docs/recipes")!)
                }
            }
            .navigationTitle("Settings")
            .confirmationDialog("Remove all favourites?", isPresented: $confirmClear, titleVisibility: .visible) {
                Button("Remove All", role: .destructive) {
                    for favorite in favorites { context.delete(favorite) }
                }
            } message: {
                Text("This can't be undone.")
            }
        }
    }
}

#Preview {
    SettingsView()
        .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}
```

## 6. The app shell

```swift title="App/RootView.swift"
import SwiftUI

struct RootView: View {
    let service: any RecipeService
    @AppStorage("appearance") private var appearance = Appearance.system
    @State private var homeViewModel: HomeViewModel

    init(service: any RecipeService) {
        self.service = service
        _homeViewModel = State(initialValue: HomeViewModel(service: service))
    }

    var body: some View {
        TabView {
            Tab("Recipes", systemImage: "fork.knife") {
                HomeView(viewModel: homeViewModel)
            }
            Tab("Favourites", systemImage: "heart") {
                FavoritesView()
            }
            Tab("Settings", systemImage: "gear") {
                SettingsView()
            }
        }
        .preferredColorScheme(appearance.colorScheme)
    }
}

#Preview {
    RootView(service: MockRecipeService())
        .modelContainer(for: FavoriteRecipe.self, inMemory: true)
}
```

```swift title="App/RecipeBoxApp.swift"
import SwiftUI
import SwiftData

@main
struct RecipeBoxApp: App {
    /// UI tests launch the app with "-ui-testing" to use mock data and a throwaway database.
    private let isUITesting = ProcessInfo.processInfo.arguments.contains("-ui-testing")

    var body: some Scene {
        WindowGroup {
            RootView(service: isUITesting ? MockRecipeService() : LiveRecipeService())
        }
        .modelContainer(for: FavoriteRecipe.self, inMemory: isUITesting)
    }
}
```

## 7. Run it! 🎉

Run the app and try everything:

- [ ] Recipes load in a grid with photos. Pull down to refresh.
- [ ] Search "pasta", "Italian" or "dessert". Clear the search, and try a word with no results.
- [ ] Open a recipe: photo, facts, ingredients, numbered steps. Share it.
- [ ] Heart recipes from the grid **and** from the detail screen; open **Favourites**; swipe to delete.
- [ ] Switch **Settings → Theme** between Light, Dark and System. The whole app follows.
- [ ] Rotate, and try an iPad simulator: more grid columns.
- [ ] Turn off your Mac's Wi-Fi and relaunch: cached recipes appear with the offline note, and favourites still open.
- [ ] Turn on **VoiceOver** (Settings → Accessibility) or use the **Accessibility Inspector**, and navigate the grid.

If something doesn't work, use the techniques from Lesson 32: breakpoints in `HomeViewModel.refresh()`, `po` in the console, and the view debugger.

:::exercise Stretch goals
1. Add a horizontal row of **meal type chips** (Breakfast, Lunch, Dinner, Dessert…) above the grid using the filter from the previous lesson's stretch goal.
2. Add a **"Surprise me"** toolbar button that navigates to a random recipe. (Hint: give the `NavigationStack` a `path`.)
3. Add a zoom transition from card to detail with `.matchedTransitionSource(id:in:)` and `.navigationTransition(.zoom(sourceID:in:))`.
:::

## Check your understanding

```quiz
Q: Why does FavoriteToggle use its own @Query instead of receiving `isFavorite` from its parent?
- [ ] @Query is faster than parameters
- [x] It makes the component self-contained, so any screen can use it without extra wiring
- [ ] Parents can't pass Booleans
- [ ] SwiftData requires it
> Self-contained components are easy to reuse; SwiftData keeps every query in sync.

Q: How do the two HomeView previews show different states?
- [ ] By editing the view's code
- [x] By injecting different MockRecipeService configurations
- [ ] By turning off the Mac's Wi-Fi
- [ ] Previews always show the same state
> Dependency injection makes every state previewable.

Q: What does `.accessibilityElement(children: .combine)` do on RecipeCard?
- [ ] Hides the card from VoiceOver
- [x] Makes VoiceOver read the card's contents as a single element
- [ ] Combines two cards
- [ ] Adds a label
> Grouping related content makes VoiceOver navigation faster.
```
