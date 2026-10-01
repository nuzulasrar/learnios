---
id: capstone-setup
title: "Capstone 1: Plan & Set Up RecipeBox"
part: 8
minutes: 30
summary: Time to build a complete app on your own. Meet RecipeBox, plan its features and architecture, create the project, and write the core Recipe model with sample data.
---

:::goals
- The features and screens of **RecipeBox**
- How the architecture maps to folders and types
- Creating and configuring the project
- The `Recipe` model, decoding the real API, and sample data for previews
:::

## Meet RecipeBox 🍳

Over the next four lessons you'll build **RecipeBox**, a complete app that uses almost everything you've learned. It loads recipes from the free [DummyJSON recipes API](https://dummyjson.com/docs/recipes), works offline, and saves favourites.

### Features

1. **Home**: a grid of recipe cards (photo, name, time, rating, cuisine) with **search** and **pull-to-refresh**.
2. **Recipe detail**: a large photo, key facts, ingredients and numbered steps, plus **favourite** and **share** buttons.
3. **Favourites**: recipes you've hearted, saved with **SwiftData**, which also work offline.
4. **Settings**: appearance (System / Light / Dark) and app info.
5. **Offline-first**: the last loaded recipes are cached on disk and shown instantly on launch.
6. **Adaptive**: more grid columns on iPad and in landscape; supports Dynamic Type and VoiceOver.

<div class="diagram"><div class="flow">
<div class="hl">Home<small>search · grid · refresh</small></div><div class="arrow">→</div>
<div>Recipe detail<small>ingredients · steps · ♥ · share</small></div>
</div>
<div class="flow" style="margin-top:8px">
<div class="hl">Favourites<small>SwiftData</small></div>
<div class="hl">Settings<small>@AppStorage</small></div>
</div>
<div class="diagram-caption">Home, Favourites and Settings are tabs; Detail is pushed onto each tab's navigation stack.</div></div>

### Architecture

<div class="diagram">
<div class="layers">
<div class="layer ui"><h5>Views</h5><div class="boxes"><span>RootView (tabs)</span><span>HomeView</span><span>RecipeDetailView</span><span>FavoritesView</span><span>SettingsView</span><span>RecipeCard</span></div></div>
<div class="dir">↓ ↑</div>
<div class="layer domain"><h5>View model</h5><div class="boxes"><span>HomeViewModel (@Observable, @MainActor)</span></div></div>
<div class="dir">↓ ↑</div>
<div class="layer data"><h5>Services & storage</h5><div class="boxes"><span>RecipeService → LiveRecipeService / MockRecipeService</span><span>RecipeCache (actor, disk)</span><span>FavoriteRecipe (SwiftData @Model)</span><span>@AppStorage</span></div></div>
</div>
</div>

### Folder structure

<div class="tree">RecipeBox/
├── App/
│   ├── RecipeBoxApp.swift
│   └── RootView.swift
├── Models/
│   ├── Recipe.swift
│   ├── Recipe+Samples.swift
│   └── FavoriteRecipe.swift
├── Services/
│   ├── RecipeService.swift
│   └── RecipeCache.swift
├── Features/
│   ├── Home/            HomeView.swift · HomeViewModel.swift
│   ├── Detail/          RecipeDetailView.swift
│   ├── Favorites/       FavoritesView.swift
│   └── Settings/        SettingsView.swift · Appearance.swift
├── Components/          RecipeCard.swift · RecipeImage.swift · FavoriteToggle.swift
└── Assets.xcassets</div>

:::tip How to approach the capstone
Try to write each file yourself **before** looking at the code in the lesson. Reuse your notes and earlier lessons. The provided code is a reference, not something to paste. If you're stuck for more than 15 minutes, peek, understand, then close it and type it yourself.
:::

## Step 1: create the project

1. **File → New → Project → iOS → App**.
2. Product Name `RecipeBox`, your Team (or None), Organization Identifier `com.yourname`, Interface **SwiftUI**, Language **Swift**, Testing System **Swift Testing**, Storage **None** (we'll add SwiftData ourselves so you see every step).
3. In the target's **General** tab, set **Minimum Deployments** to **iOS 18.0**.
4. In **Assets.xcassets**, set **AccentColor** to a warm orange (e.g. `#F05138`) for both appearances.
5. Delete `ContentView.swift`; we'll create `RootView` instead. (Xcode will show an error in `RecipeBoxApp.swift` until then.)
6. Create the folders: right-click the `RecipeBox` folder → **New Folder** for `App`, `Models`, `Services`, `Features` (with `Home`, `Detail`, `Favorites`, `Settings` inside), and `Components`. Move `RecipeBoxApp.swift` into `App`.

## Step 2: look at the API

Open [dummyjson.com/recipes/1](https://dummyjson.com/recipes/1) in your browser:

```json
{
  "id": 1,
  "name": "Classic Margherita Pizza",
  "ingredients": ["Pizza dough", "Tomato sauce", "Fresh mozzarella cheese", "…"],
  "instructions": ["Preheat the oven to 475°F (245°C).", "…"],
  "prepTimeMinutes": 20,
  "cookTimeMinutes": 15,
  "servings": 4,
  "difficulty": "Easy",
  "cuisine": "Italian",
  "caloriesPerServing": 300,
  "tags": ["Pizza", "Italian"],
  "userId": 166,
  "image": "https://cdn.dummyjson.com/recipe-images/1.webp",
  "rating": 4.6,
  "reviewCount": 98,
  "mealType": ["Dinner"]
}
```

`https://dummyjson.com/recipes?limit=0` returns all ~50 recipes as `{ "recipes": [ … ], "total": 50, … }`.

## Step 3: the Recipe model

```swift title="Models/Recipe.swift"
import Foundation

struct Recipe: Codable, Identifiable, Hashable, Sendable {
    let id: Int
    let name: String
    let ingredients: [String]
    let instructions: [String]
    let prepTimeMinutes: Int
    let cookTimeMinutes: Int
    let servings: Int
    let difficulty: String
    let cuisine: String
    let caloriesPerServing: Int
    let tags: [String]
    let image: String
    let rating: Double
    let mealType: [String]

    var totalMinutes: Int { prepTimeMinutes + cookTimeMinutes }
    var imageURL: URL? { URL(string: image) }

    /// Plain-text version for sharing.
    var shareText: String {
        """
        \(name) (\(cuisine), \(totalMinutes) min)

        Ingredients:
        \(ingredients.map { "• \($0)" }.joined(separator: "\n"))
        """
    }
}

/// The shape of `GET /recipes`.
struct RecipeListResponse: Codable, Sendable {
    let recipes: [Recipe]
}
```

- Property names match the JSON keys exactly, so Codable needs no `CodingKeys`. Keys we don't need (`userId`, `reviewCount`) are simply ignored.
- It's `Hashable` so we can push it with `NavigationLink(value:)`, and `Sendable` so it can move safely between actors.
- `totalMinutes`, `imageURL` and `shareText` are **computed**, derived from the stored data.

## Step 4: sample data for previews and tests

Previews and tests shouldn't hit the network. Give them realistic sample recipes:

```swift title="Models/Recipe+Samples.swift"
import Foundation

extension Recipe {
    static let sample = Recipe(
        id: 1,
        name: "Classic Margherita Pizza",
        ingredients: ["Pizza dough", "Tomato sauce", "Fresh mozzarella cheese", "Fresh basil leaves", "Olive oil"],
        instructions: [
            "Preheat the oven to 475°F (245°C).",
            "Roll out the pizza dough and spread tomato sauce evenly.",
            "Top with slices of fresh mozzarella and basil leaves.",
            "Bake for 12–15 minutes until the crust is golden."
        ],
        prepTimeMinutes: 20, cookTimeMinutes: 15, servings: 4,
        difficulty: "Easy", cuisine: "Italian", caloriesPerServing: 300,
        tags: ["Pizza", "Italian"],
        image: "https://cdn.dummyjson.com/recipe-images/1.webp",
        rating: 4.6, mealType: ["Dinner"]
    )

    static let samples: [Recipe] = [
        sample,
        Recipe(id: 2, name: "Vegetarian Stir-Fry", ingredients: ["Tofu", "Broccoli", "Soy sauce"],
               instructions: ["Stir-fry the vegetables.", "Add tofu and sauce."],
               prepTimeMinutes: 15, cookTimeMinutes: 20, servings: 3, difficulty: "Medium",
               cuisine: "Asian", caloriesPerServing: 250, tags: ["Vegetarian", "Stir-fry"],
               image: "https://cdn.dummyjson.com/recipe-images/2.webp", rating: 4.7, mealType: ["Lunch"]),
        Recipe(id: 3, name: "Chocolate Chip Cookies", ingredients: ["Butter", "Sugar", "Chocolate chips"],
               instructions: ["Cream butter and sugar.", "Fold in chips and bake."],
               prepTimeMinutes: 15, cookTimeMinutes: 10, servings: 24, difficulty: "Easy",
               cuisine: "American", caloriesPerServing: 150, tags: ["Cookies", "Dessert"],
               image: "https://cdn.dummyjson.com/recipe-images/3.webp", rating: 4.9, mealType: ["Snack", "Dessert"])
    ]
}
```

(`sample` is defined inside `extension Recipe`, so `samples` can refer to it directly.)

## Step 5: a temporary root view

So the project builds, create a placeholder we'll replace in Lesson 36:

```swift title="App/RootView.swift (temporary)" nocheck
import SwiftUI

struct RootView: View {
    var body: some View {
        List(Recipe.samples) { recipe in
            LabeledContent(recipe.name, value: "\(recipe.totalMinutes) min")
        }
    }
}

#Preview { RootView() }
```

```swift title="App/RecipeBoxApp.swift (temporary)" nocheck
import SwiftUI

@main
struct RecipeBoxApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
    }
}
```

Build and run ([[Cmd]] + [[R]]). You should see three sample recipes. Commit this to Git (**Integrate → Commit**, or `git commit` in Terminal): *"Project setup and Recipe model"*. Commit at the end of each capstone lesson so you can always go back.

:::recap
- RecipeBox: Home (search, grid, refresh), Detail, Favourites (SwiftData), Settings; offline-first.
- Views → `HomeViewModel` → `RecipeService` + `RecipeCache` + SwiftData.
- `Recipe` is a `Codable`, `Hashable`, `Sendable` struct matching the API, with computed helpers.
- Sample data powers previews and tests.
:::

## Check your understanding

```quiz
Q: Why does Recipe conform to Hashable?
- [ ] To decode JSON
- [x] So it can be used as a navigation value with NavigationLink(value:)
- [ ] To save it to disk
- [ ] It's required by Identifiable
> Navigation values must be Hashable.

Q: The API includes `userId` and `reviewCount`, but Recipe doesn't declare them. What happens when decoding?
- [ ] Decoding fails
- [x] They're ignored
- [ ] They're stored in a dictionary
- [ ] The app crashes
> Codable ignores unknown keys.

Q: Why have `Recipe.samples`?
- [ ] To show on the App Store
- [x] So previews and tests work instantly and offline, without the network
- [ ] Because the API is slow
- [ ] They're required by SwiftUI
> Sample data keeps development fast and deterministic.
```
