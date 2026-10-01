---
id: navigation
title: Navigation, Tabs & Sheets
part: 4
minutes: 40
summary: Move between screens the iOS way. Covers NavigationStack with value-based links, programmatic navigation, toolbars, tab bars, sheets and dismissing.
---

:::goals
- `NavigationStack`, `NavigationLink` and `navigationDestination`
- Programmatic navigation with a path
- Titles and toolbars
- Tab bars with `TabView` and `Tab`
- Sheets, full-screen covers and `dismiss`
- `NavigationSplitView` for iPad (overview)
:::

## `NavigationStack`

A `NavigationStack` manages a **stack** of screens: push a new screen on top, and the back button or swipe gesture pops it off.

<div class="diagram"><div class="flow">
<div>Recipes</div><div class="arrow">→</div><div>Recipe detail</div><div class="arrow">→</div><div class="hl">Step-by-step<small>top of the stack</small></div>
</div><div class="diagram-caption">Swiping from the left edge (or tapping Back) pops the top screen.</div></div>

### Value-based navigation

The modern pattern: links push a **value**, and `navigationDestination(for:)` decides which screen shows that value:

```swift title="RecipesNavigation.swift"
import SwiftUI

struct Recipe: Identifiable, Hashable {
    let id: Int
    let name: String
    let minutes: Int
}

struct RecipeListView: View {
    let recipes = [
        Recipe(id: 1, name: "Pancakes", minutes: 20),
        Recipe(id: 2, name: "Ramen", minutes: 45),
        Recipe(id: 3, name: "Tacos", minutes: 30)
    ]

    var body: some View {
        NavigationStack {
            List(recipes) { recipe in
                NavigationLink(value: recipe) {          // push this value
                    Text(recipe.name)
                }
            }
            .navigationTitle("Recipes")
            .navigationDestination(for: Recipe.self) { recipe in   // how to show a Recipe
                RecipeDetailView(recipe: recipe)
            }
        }
    }
}

struct RecipeDetailView: View {
    let recipe: Recipe

    var body: some View {
        VStack(spacing: 12) {
            Text(recipe.name).font(.largeTitle.bold())
            Text("Ready in \(recipe.minutes) minutes").foregroundStyle(.secondary)
        }
        .navigationTitle(recipe.name)
        .navigationBarTitleDisplayMode(.inline)
    }
}

#Preview { RecipeListView() }
```

- The value must be **`Hashable`**.
- Put `navigationDestination` **inside** the stack (usually on the root view), not inside a lazy container like a `List` row.
- There's only **one** `NavigationStack`, at the root. Detail screens don't need their own.

## Programmatic navigation: the path

Bind the stack to a **path**, an array of values, and you can navigate from code: jump deep, go back several screens, or pop to the root.

```swift title="PathNavigation.swift"
import SwiftUI

struct PathNavigation: View {
    @State private var path: [Int] = []             // the stack's contents

    var body: some View {
        NavigationStack(path: $path) {
            VStack(spacing: 16) {
                Button("Open step 1") { path.append(1) }
                Button("Jump straight to step 3") { path = [1, 2, 3] }
            }
            .navigationTitle("Start")
            .navigationDestination(for: Int.self) { step in
                VStack(spacing: 16) {
                    Text("Step \(step)").font(.largeTitle)
                    Button("Next step") { path.append(step + 1) }
                    Button("Back to start") { path.removeAll() }   // pop to root
                }
                .navigationTitle("Step \(step)")
            }
        }
    }
}

#Preview { PathNavigation() }
```

For stacks with *different* value types, use `NavigationPath`, a type-erased path: `@State private var path = NavigationPath()`.

:::tip Navigation state belongs to you
Because the path is plain state, you can save it, restore it, or drive it from a view model, for example to navigate after a successful save or when a notification is tapped.
:::

## Titles and toolbars

```swift
List { … }
    .navigationTitle("Recipes")                   // large title by default
    .toolbar {
        ToolbarItem(placement: .topBarTrailing) {
            Button("Add", systemImage: "plus") { showNewRecipe = true }
        }
        ToolbarItem(placement: .topBarLeading) {
            EditButton()
        }
    }
    .searchable(text: $searchText, prompt: "Search recipes")   // search field in the nav bar
```

## Tab bars

A `TabView` shows top-level sections. Each tab usually contains its own `NavigationStack`:

```swift title="MainTabs.swift"
import SwiftUI

struct MainTabs: View {
    var body: some View {
        TabView {
            Tab("Home", systemImage: "house") {
                NavigationStack {
                    Text("Home").navigationTitle("Home")
                }
            }
            Tab("Favourites", systemImage: "heart") {
                NavigationStack {
                    Text("Favourites").navigationTitle("Favourites")
                }
            }
            .badge(3)
            Tab("Settings", systemImage: "gear") {
                NavigationStack {
                    Text("Settings").navigationTitle("Settings")
                }
            }
            Tab(role: .search) {                 // a dedicated search tab
                NavigationStack {
                    Text("Search").navigationTitle("Search")
                }
            }
        }
    }
}

#Preview { MainTabs() }
```

Each tab keeps its own navigation state, so switching tabs and coming back returns you to where you were. To select tabs from code, pass `TabView(selection: $selectedTab)` and give each `Tab` a `value:`.

## Sheets and full-screen covers

A **sheet** slides up over the current screen for a focused task, like creating a new item or showing settings. Like alerts, it's driven by state:

```swift title="SheetDemo.swift"
import SwiftUI

struct NewRecipeView: View {
    @Environment(\.dismiss) private var dismiss      // closes the sheet
    @State private var name = ""
    let onSave: (String) -> Void

    var body: some View {
        NavigationStack {
            Form {
                TextField("Recipe name", text: $name)
            }
            .navigationTitle("New Recipe")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Cancel") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") {
                        onSave(name)
                        dismiss()
                    }
                    .disabled(name.isEmpty)
                }
            }
        }
    }
}

struct SheetDemo: View {
    @State private var showNew = false
    @State private var names: [String] = []

    var body: some View {
        NavigationStack {
            List(names, id: \.self) { Text($0) }
                .navigationTitle("My Recipes")
                .toolbar {
                    Button("Add", systemImage: "plus") { showNew = true }
                }
                .sheet(isPresented: $showNew) {
                    NewRecipeView { names.append($0) }
                        .presentationDetents([.medium, .large])     // half-height sheet
                }
        }
    }
}

#Preview { SheetDemo() }
```

- `.sheet(item: $selectedRecipe) { recipe in … }` shows a sheet for an optional item: it's presented when the item is set and dismissed when it's `nil`.
- `.fullScreenCover(isPresented:)` covers the whole screen, which suits onboarding or a camera.
- `@Environment(\.dismiss)` closes the current sheet **or** pops the current pushed screen.

## iPad: `NavigationSplitView`

On iPad and Mac, a sidebar + detail layout is common. `NavigationSplitView` provides it, and automatically collapses into a stack on iPhone:

```swift
NavigationSplitView {
    List(categories, selection: $selectedCategory) { … }     // sidebar
} detail: {
    RecipeGrid(category: selectedCategory)                   // detail
}
```

## Recap

:::recap
- One `NavigationStack` at the root; `NavigationLink(value:)` + `.navigationDestination(for:)`.
- Bind a `path` for programmatic navigation: append to push, `removeAll()` to pop to root.
- `.navigationTitle`, `.toolbar`, `.searchable` configure the navigation bar.
- `TabView` + `Tab` for top-level sections, each with its own stack.
- `.sheet` / `.fullScreenCover` are state-driven; `dismiss` closes them.
:::

## Practice

:::exercise
Build a small three-level app:

1. A **Categories** list (Breakfast, Lunch, Dinner) inside a `NavigationStack`.
2. Tapping a category pushes a list of 5 made-up recipes for it.
3. Tapping a recipe pushes a detail screen with a **Done cooking** button that pops all the way back to Categories. (Hint: use a `NavigationPath` because the values have different types.)
4. Add a toolbar **+** button on Categories that opens a sheet.
:::

:::solution Show solution
```swift
import SwiftUI

struct Category: Hashable { let name: String }
struct Dish: Hashable { let name: String; let category: String }

struct CookbookView: View {
    @State private var path = NavigationPath()
    @State private var showAbout = false
    let categories = ["Breakfast", "Lunch", "Dinner"].map(Category.init)

    var body: some View {
        NavigationStack(path: $path) {
            List(categories, id: \.self) { category in
                NavigationLink(category.name, value: category)
            }
            .navigationTitle("Categories")
            .toolbar {
                Button("About", systemImage: "plus") { showAbout = true }
            }
            .navigationDestination(for: Category.self) { category in
                List(1...5, id: \.self) { i in
                    NavigationLink("\(category.name) idea \(i)",
                                   value: Dish(name: "\(category.name) idea \(i)", category: category.name))
                }
                .navigationTitle(category.name)
            }
            .navigationDestination(for: Dish.self) { dish in
                VStack(spacing: 20) {
                    Text(dish.name).font(.title)
                    Button("Done cooking") { path = NavigationPath() }   // pop to root
                        .buttonStyle(.borderedProminent)
                }
                .navigationTitle(dish.category)
            }
            .sheet(isPresented: $showAbout) {
                Text("New category coming soon!").presentationDetents([.medium])
            }
        }
    }
}

#Preview { CookbookView() }
```
:::

## Check your understanding

```quiz
Q: What must a value pushed with `NavigationLink(value:)` conform to?
- [ ] Codable
- [x] Hashable
- [ ] Identifiable only
- [ ] View
> Navigation values are stored in the path, which requires Hashable.

Q: How do you pop to the root of a NavigationStack bound to `path: [Int]`?
- [ ] dismiss()
- [x] path.removeAll()
- [ ] path.append(0)
- [ ] Recreate the NavigationStack
> The path is the stack; emptying it shows the root.

Q: A detail screen pushed onto a stack needs its own NavigationStack. True?
- [ ] True
- [x] False. There's one stack at the root; pushed screens are part of it
- [ ] Only on iPad
- [ ] Only if it has a toolbar
> Nested NavigationStacks cause confusing, broken navigation.

Q: What does `@Environment(\.dismiss)` do when called in a sheet?
- [ ] Deletes the sheet's data
- [x] Closes the sheet
- [ ] Pops to the root of the app
- [ ] Nothing, it's only for alerts
> dismiss closes the current presentation, or pops the current pushed screen.
```
