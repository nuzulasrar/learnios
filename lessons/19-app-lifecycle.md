---
id: app-lifecycle
title: The App Lifecycle & Scenes
part: 4
minutes: 25
summary: Understand how iOS launches, backgrounds and terminates your app, react to scene phases, run work when views appear, and restore the user's place.
---

:::goals
- The `App` protocol, scenes and `WindowGroup`
- App states: active, inactive, background, suspended, terminated
- Reacting to `scenePhase`
- `onAppear`, `onDisappear` and `.task`
- Restoring state with `@SceneStorage`
- When you still need an app delegate
:::

## The app's structure

```swift title="RecipeBoxApp.swift"
import SwiftUI

@main
struct RecipeBoxApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

struct ContentView: View {
    var body: some View { Text("Hello") }
}
```

- The **`App`** is created once when your app launches.
- A **`Scene`** is a part of your app's UI that the system manages. `WindowGroup` creates windows showing your root view. On iPad and Mac the user can open several windows (several scenes) of the same app side by side.

## App states

iOS, not your app, decides when your app runs. Your app moves through these states:

<div class="diagram"><div class="flow">
<div>Not running</div><div class="arrow">→</div>
<div class="hl">Active<small>on screen, receiving touches</small></div><div class="arrow">⇄</div>
<div>Inactive<small>visible but not interactive (e.g. Control Centre open)</small></div><div class="arrow">⇄</div>
<div>Background<small>not visible, briefly running</small></div><div class="arrow">→</div>
<div>Suspended<small>in memory, not running</small></div>
</div><div class="diagram-caption">A suspended app can be terminated at any time to free memory, without warning.</div></div>

When the user leaves your app, it gets a few seconds in the background, then it's **suspended**: frozen in memory. If the system needs memory, it **terminates** suspended apps silently. When the user comes back, they expect to be exactly where they left off, so save important state before you're suspended.

## `scenePhase`

Read the current phase from the environment and react to changes:

```swift title="PhaseAwareView.swift"
import SwiftUI

struct PhaseAwareView: View {
    @Environment(\.scenePhase) private var scenePhase
    @State private var log: [String] = []

    var body: some View {
        List(log, id: \.self) { Text($0) }
            .onChange(of: scenePhase) { _, newPhase in
                switch newPhase {
                case .active: log.append("Active: refresh data")
                case .inactive: log.append("Inactive")
                case .background: log.append("Background: save drafts now")
                @unknown default: break
                }
            }
    }
}

#Preview { PhaseAwareView() }
```

Typical uses:

- **`.background`**: save unsaved work, pause timers, stop expensive work.
- **`.active`**: refresh data that may be stale, resume timers.

## View lifecycle: `onAppear`, `onDisappear` and `.task`

```swift
List(recipes) { … }
    .onAppear { print("List appeared") }        // synchronous
    .onDisappear { print("List disappeared") }
    .task {                                     // async; cancelled when the view disappears
        await viewModel.load()
    }
    .task(id: searchText) {                     // restarts whenever searchText changes
        await viewModel.search(searchText)
    }
```

Prefer **`.task`** for async loading. It's tied to the view's lifetime, so if the user leaves the screen before loading finishes, the work is cancelled automatically. `.task(id:)` is a neat way to re-run work when a value changes, like a search that cancels the previous one.

:::warning onAppear can run more than once
Views can appear several times, for example when navigating back to them or switching tabs. Make loading code idempotent: check whether data is already loaded, or make reloading harmless.
:::

## Restoring the user's place: `@SceneStorage`

`@SceneStorage` saves small values per scene and restores them after the app is terminated and relaunched, which is perfect for the selected tab or a draft:

```swift title="DraftView.swift"
import SwiftUI

struct DraftView: View {
    @SceneStorage("selectedTab") private var selectedTab = 0
    @SceneStorage("draftNote") private var draft = ""

    var body: some View {
        TabView(selection: $selectedTab) {
            TextEditor(text: $draft)
                .padding()
                .tabItem { Label("Draft", systemImage: "square.and.pencil") }
                .tag(0)
            Text("Settings")
                .tabItem { Label("Settings", systemImage: "gear") }
                .tag(1)
        }
    }
}

#Preview { DraftView() }
```

Use it only for small UI state. Real data belongs in SwiftData or files (Part 5).

:::tip Test termination
Run the app from Xcode, type a draft, press [[Cmd]] + [[Shift]] + [[H]] to go home, then stop the app with Xcode's ■ button (this simulates the system terminating it). Run again: your draft and tab should be restored.
:::

## Environment values

`scenePhase` is one of many **environment values** SwiftUI provides to every view:

```swift
@Environment(\.colorScheme) private var colorScheme           // .light / .dark
@Environment(\.dynamicTypeSize) private var typeSize
@Environment(\.horizontalSizeClass) private var sizeClass     // .compact (iPhone portrait) / .regular (iPad)
@Environment(\.locale) private var locale
@Environment(\.dismiss) private var dismiss                    // close a sheet or pop a screen
@Environment(\.openURL) private var openURL
```

`horizontalSizeClass` is handy for adapting layouts to iPad: show a grid with more columns, or a side-by-side layout.

## When you need an app delegate

Some system events still arrive through the older `UIApplicationDelegate`, for example registering for **remote push notifications** or configuring certain SDKs at launch. Add one with an adaptor:

```swift
final class AppDelegate: NSObject, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        // one-time setup
        return true
    }
}

@main
struct RecipeBoxApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

Most apps never need one.

## Recap

:::recap
- `App` → `Scene` (`WindowGroup`) → views.
- iOS moves apps between active, inactive, background and suspended, and can terminate suspended apps without warning.
- React to `scenePhase` to save work in the background and refresh when active.
- `.task` for async work tied to a view; `.task(id:)` re-runs when a value changes.
- `@SceneStorage` restores small UI state after termination.
:::

:::exercise
1. Add the `PhaseAwareView` to a project, run it, and trigger each phase: pull down Control Centre (inactive), go Home (background), come back (active).
2. Build a note-taking screen with `@SceneStorage` for the draft, and verify it survives termination using the tip above.
:::

## Check your understanding

```quiz
Q: What can happen to a suspended app without any warning?
- [ ] It becomes active
- [x] The system can terminate it to free memory
- [ ] It keeps running network requests forever
- [ ] It's uninstalled
> Save important state when entering the background.

Q: Why is `.task` preferred over `onAppear` for loading data?
- [ ] It runs faster
- [x] It supports async work and is cancelled automatically when the view disappears
- [ ] It runs only once per app launch
- [ ] It runs on a background thread
> .task ties async work to the view's lifetime.

Q: What is `@SceneStorage` good for?
- [ ] Storing a large database
- [x] Restoring small UI state, like the selected tab or a draft, after the app is terminated and relaunched
- [ ] Sharing data between apps
- [ ] Storing passwords
> It's lightweight state restoration per scene.

Q: Which environment value tells you whether you have iPad-like horizontal space?
- [ ] colorScheme
- [ ] scenePhase
- [x] horizontalSizeClass
- [ ] locale
> .regular width usually means iPad (or a large iPhone in landscape).
```
