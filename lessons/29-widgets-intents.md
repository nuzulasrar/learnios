---
id: widgets-intents
title: Widgets, App Intents & Live Activities
part: 6
minutes: 30
summary: Put your app on the Home Screen, Lock Screen, Siri and Spotlight. Build a WidgetKit widget with a timeline, share data with an App Group, and expose actions with App Intents.
---

:::goals
- What widgets are and how their timelines work
- Adding a widget extension and building a widget with SwiftUI
- Sharing data between the app and the widget with an App Group
- Interactive widgets and App Intents for Siri, Shortcuts and Spotlight
- An overview of Live Activities
:::

## Widgets in one minute

**Widgets** show glanceable information from your app on the Home Screen, the Lock Screen, in StandBy and in Control Centre. They're built with **SwiftUI** and **WidgetKit**, but they aren't mini-apps that run constantly: your widget provides a **timeline** of snapshots in advance, and iOS displays them at the right times.

<div class="diagram"><div class="flow">
<div>TimelineProvider<small>creates entries</small></div><div class="arrow">→</div>
<div class="hl">Timeline<small>entry @ 9:00, entry @ 12:00, …</small></div><div class="arrow">→</div>
<div>Widget view<small>SwiftUI renders each entry</small></div>
</div></div>

## 1. Add a widget extension

**File → New → Target… → Widget Extension**, name it `RecipeWidget`, untick *Include Configuration App Intent* for now, and click **Activate** when Xcode asks about the scheme. Xcode creates a new target with a template widget.

## 2. The widget code

```swift title="RecipeWidget.swift"
import WidgetKit
import SwiftUI

// 1. The data for one moment in time
struct RecipeEntry: TimelineEntry {
    let date: Date
    let recipeName: String
    let minutes: Int
}

// 2. Provides entries to WidgetKit
struct RecipeProvider: TimelineProvider {
    func placeholder(in context: Context) -> RecipeEntry {       // shown while loading (redacted)
        RecipeEntry(date: .now, recipeName: "Recipe of the day", minutes: 30)
    }

    func getSnapshot(in context: Context, completion: @escaping (RecipeEntry) -> Void) {
        completion(RecipeEntry(date: .now, recipeName: "Margherita Pizza", minutes: 35))   // widget gallery
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<RecipeEntry>) -> Void) {
        let ideas = [("Margherita Pizza", 35), ("Pad Thai", 25), ("Pancakes", 20)]
        let now = Date.now
        // One entry every 4 hours, then ask for a new timeline
        let entries = ideas.enumerated().map { index, idea in
            RecipeEntry(date: now.addingTimeInterval(Double(index) * 4 * 3600),
                        recipeName: idea.0, minutes: idea.1)
        }
        completion(Timeline(entries: entries, policy: .atEnd))
    }
}

// 3. The SwiftUI view
struct RecipeWidgetView: View {
    let entry: RecipeEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            Label("Today", systemImage: "fork.knife")
                .font(.caption.bold())
                .foregroundStyle(.orange)
            Text(entry.recipeName)
                .font(.headline)
                .lineLimit(2)
            Spacer()
            Label("\(entry.minutes) min", systemImage: "clock")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .containerBackground(.fill.tertiary, for: .widget)
    }
}

// 4. The widget's configuration
struct RecipeWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "RecipeWidget", provider: RecipeProvider()) { entry in
            RecipeWidgetView(entry: entry)
        }
        .configurationDisplayName("Recipe of the Day")
        .description("A new cooking idea a few times a day.")
        .supportedFamilies([.systemSmall, .systemMedium, .accessoryRectangular])
    }
}
```

The template also includes a `@main` **WidgetBundle** listing your widgets. Run the **RecipeWidget** scheme to see it in the Simulator, or use the canvas: widgets support `#Preview(as: .systemSmall) { RecipeWidget() } timeline: { … }`.

:::note Widgets are snapshots
Widget views can't have timers, scroll views or live network calls. Keep timelines reasonable: iOS gives each widget a daily refresh budget. When your app's data changes, call `WidgetCenter.shared.reloadAllTimelines()` to ask for an update.
:::

## 3. Share data with an App Group

The app and the widget are separate processes with separate sandboxes. To share data (say, the user's favourite recipes):

1. Add the **App Groups** capability to **both** targets with the same group, e.g. `group.com.yourname.recipebox`.
2. Read and write shared storage:

```swift
let shared = UserDefaults(suiteName: "group.com.yourname.recipebox")
shared?.set("Pad Thai", forKey: "lastViewedRecipe")

let folder = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.yourname.recipebox")
```

SwiftData can also use the shared container, so the widget can query the same store.

## 4. Tapping a widget

Tapping a widget opens your app. Use `.widgetURL(URL(string: "recipebox://recipe/42"))` on the widget view, and your `onOpenURL` handler (Lesson 27) navigates to the right screen. Medium and large widgets can have several `Link`s to different destinations.

## App Intents: Siri, Shortcuts, Spotlight & interactive widgets

An **App Intent** exposes an action from your app to the system: Siri, the Shortcuts app, Spotlight, the Action button, and **buttons inside widgets**.

```swift title="StartTimerIntent.swift" nocheck
import AppIntents

struct StartTimerIntent: AppIntent {
    static let title: LocalizedStringResource = "Start Cooking Timer"
    static let description = IntentDescription("Starts a cooking timer in RecipeBox.")

    @Parameter(title: "Minutes", default: 10)
    var minutes: Int

    func perform() async throws -> some IntentResult & ProvidesDialog {
        try await Reminders.timer(minutes: Double(minutes), recipeName: "Your dish")
        return .result(dialog: "Timer set for \(minutes) minutes.")
    }
}

struct RecipeShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(intent: StartTimerIntent(),
                    phrases: ["Start a cooking timer in \(.applicationName)"],
                    shortTitle: "Cooking Timer",
                    systemImageName: "timer")
    }
}
```

With `AppShortcutsProvider`, users can say *"Start a cooking timer in RecipeBox"* to Siri with **zero setup**, and the action also appears in Spotlight and the Shortcuts app. In a widget, `Button(intent: StartTimerIntent()) { … }` makes the widget interactive. App Intents are also how your app's actions become available to Apple Intelligence.

## Live Activities (overview)

**Live Activities** show real-time, ongoing information on the Lock Screen and in the **Dynamic Island**: a delivery on its way, a sports score, a cooking timer counting down. They're built with **ActivityKit** plus a widget extension:

1. Define `ActivityAttributes` with the static data and a `ContentState` for what changes.
2. Build the Lock Screen and Dynamic Island layouts in your widget extension with `ActivityConfiguration`.
3. Start it from the app with `Activity.request(...)`, update it with `activity.update(...)`, and end it with `activity.end(...)`. Updates can also arrive by push.

A countdown timer is a perfect first Live Activity, since `Text(timerInterval:)` counts down by itself without any updates.

## Recap

:::recap
- Widgets are SwiftUI snapshots from a `TimelineProvider`, added as a Widget Extension target.
- Share data with an **App Group**; reload timelines when data changes.
- `widgetURL` / `Link` deep-link into your app.
- **App Intents** expose actions to Siri, Shortcuts, Spotlight and interactive widgets.
- Live Activities show ongoing events on the Lock Screen and Dynamic Island.
:::

:::exercise
1. Add the Recipe of the Day widget to one of your projects, and make it deep-link into the app.
2. Add an App Group and make the widget show the name of the last recipe the user viewed in the app. Call `WidgetCenter.shared.reloadAllTimelines()` when it changes.
3. Create an App Intent "Show a random recipe" and try it from the Shortcuts app.
:::

## Check your understanding

```quiz
Q: How does a widget get its content?
- [ ] It runs continuously like a small app
- [x] A TimelineProvider supplies a timeline of entries that iOS renders at the right times
- [ ] It streams video of the app
- [ ] It downloads HTML
> Widgets are snapshots rendered from timeline entries.

Q: How can the app and its widget share data?
- [ ] They share the same sandbox automatically
- [x] Through an App Group container (shared UserDefaults, files or SwiftData store)
- [ ] Through the clipboard
- [ ] They can't
> App Groups give both targets a shared container.

Q: What do App Intents let you do?
- [ ] Make network requests faster
- [x] Expose your app's actions to Siri, Shortcuts, Spotlight and interactive widgets
- [ ] Replace SwiftUI views
- [ ] Send push notifications
> Intents describe actions the system can run on your app's behalf.

Q: After the user changes their favourites in the app, how do you update the widget?
- [ ] Wait for the next day
- [x] Call `WidgetCenter.shared.reloadAllTimelines()`
- [ ] Restart the phone
- [ ] Widgets update themselves every second
> Reloading asks WidgetKit for a fresh timeline.
```
