---
id: swiftui-intro
title: Thinking in SwiftUI
part: 3
minutes: 30
summary: SwiftUI is Apple's modern, declarative UI framework. Learn the mindset, how views and modifiers work, how SwiftUI updates the screen when state changes, and how to use previews.
---

:::goals
- Imperative vs. declarative UI, and why SwiftUI is declarative
- Views as lightweight structs with a `body`
- How modifiers work and why they return new views
- Your first interactive view with `@State`
- Composing screens from small views, and previews
:::

## Imperative vs. declarative

In UIKit, the older framework, you build views once and then **change them manually** whenever data changes:

```swift
// Imperative (UIKit)
@objc func buttonTapped() {
    count += 1
    label.text = "Tapped \(count) times"      // YOU must remember to update every label
    label.textColor = count > 5 ? .red : .label
}
```

As apps grow, keeping every view in sync with the data becomes a major source of bugs.

**SwiftUI is declarative**: you describe what the UI should look like **for the current state**, and SwiftUI keeps the screen in sync whenever the state changes.

```swift
// Declarative (SwiftUI)
Text("Tapped \(count) times")
    .foregroundStyle(count > 5 ? .red : .primary)
```

<div class="diagram"><div class="flow">
<div class="hl">State<small>your data</small></div><div class="arrow">→</div>
<div>body<small>UI = f(state)</small></div><div class="arrow">→</div>
<div class="hl">Screen</div>
</div><div class="diagram-caption">Change the state, and SwiftUI recomputes the body and updates only what changed.</div></div>

:::analogy
Imperative UI is giving turn-by-turn directions. Declarative UI is giving the destination address: you say **what** you want, and SwiftUI works out **how** to get the screen there.
:::

## Views are structs

A SwiftUI view is a small struct that conforms to `View` and has a `body`:

```swift title="GreetingView.swift"
import SwiftUI

struct GreetingView: View {
    let name: String                      // data it needs, passed in

    var body: some View {
        Text("Hello, \(name)!")
            .font(.title)
    }
}

#Preview {
    GreetingView(name: "Ada")
}
```

Things to notice:

- Views are **cheap values**, not heavy objects. SwiftUI creates and throws them away constantly. Don't worry about "creating too many views".
- `body` is a **computed property**. SwiftUI calls it whenever it needs to know what to display, so keep it fast and free of side effects (no network calls in `body`!).
- `some View` means "one specific view type" (Lesson 9). You don't need to know the exact type.

## Composing views

Build screens by **combining small views**:

```swift title="WelcomeView.swift"
import SwiftUI

struct WelcomeView: View {
    var body: some View {
        VStack(spacing: 12) {
            Image(systemName: "sparkles")
                .font(.system(size: 48))
                .foregroundStyle(.orange)
            GreetingView(name: "Ada")
            GreetingView(name: "Grace")
            Text("Let's build something great.")
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}

#Preview {
    WelcomeView()
}
```

The `VStack { … }` part uses a **result builder**: inside the braces you just list views, and SwiftUI assembles them. You can also use `if`, `if let`, `switch` and `ForEach` inside a view builder:

```swift
VStack {
    if isLoggedIn {
        Text("Welcome back!")
    } else {
        Button("Sign in") { signIn() }
    }
}
```

## Modifiers

**Modifiers** customise a view: font, colour, padding, background, frame and so on. Each modifier **returns a new view** that wraps the original:

```swift
Text("Swift")
    .font(.largeTitle.bold())
    .foregroundStyle(.white)
    .padding()
    .background(.orange)
    .clipShape(.rect(cornerRadius: 12))
```

Because each modifier wraps the previous result, **order matters**:

```swift title="ModifierOrderView.swift"
import SwiftUI

struct ModifierOrderView: View {
    var body: some View {
        VStack(spacing: 20) {
            Text("Padding, then background")
                .padding()
                .background(.orange)          // orange covers the padding too

            Text("Background, then padding")
                .background(.orange)          // orange hugs the text only
                .padding()
        }
    }
}

#Preview { ModifierOrderView() }
```

:::tip Discover modifiers
Press [[Cmd]] + [[Shift]] + [[L]] to open the Library and browse views and modifiers, or select a view in the canvas and use the **Attributes inspector** on the right. Xcode writes the code for you, and you can learn from what it writes.
:::

## Your first interactive view: `@State`

```swift title="CounterView.swift"
import SwiftUI

struct CounterView: View {
    @State private var count = 0              // state SwiftUI watches

    var body: some View {
        VStack(spacing: 16) {
            Text("You tapped \(count) times")
                .font(.title2)
                .foregroundStyle(count > 10 ? .red : .primary)

            Button("Tap me") {
                count += 1                    // change state → body re-runs → screen updates
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}

#Preview { CounterView() }
```

What happens when you tap:

1. `count += 1` changes the **state**.
2. SwiftUI notices that `body` reads `count` and re-evaluates `body`.
3. It compares the new description with the old one and updates only the `Text` that changed.

You never told the label to update. You changed the state, and the UI followed. ✨ We'll dig into `@State` and data flow in Lesson 15.

:::note Why can't I just use `var count = 0`?
A view struct is recreated all the time, so a plain property would reset, and structs can't change their own properties from inside `body` anyway. `@State` stores the value *outside* the struct, in storage SwiftUI manages, and tells SwiftUI to refresh when it changes.
:::

## Previews

`#Preview` shows your view in Xcode's canvas. You can have several, with different data and settings:

```swift
#Preview("Short name") {
    GreetingView(name: "Al")
}

#Preview("Long name, dark") {
    GreetingView(name: "Maximilian Alexander Featherstonehaugh")
        .preferredColorScheme(.dark)
}
```

Use the controls under the canvas to try **Dynamic Type sizes**, **orientation**, **dark mode** and different **devices**. Previews are live, so buttons work in them.

## Extracting subviews

When `body` gets long, extract parts into their own views. Right-click a view in the editor and choose **Refactor → Extract Subview**, or write it by hand:

```swift title="ProfileCard.swift"
import SwiftUI

struct ProfileCard: View {
    let name: String
    let role: String
    let isFollowing: Bool
    let onFollow: () -> Void          // an event it reports to its parent

    var body: some View {
        HStack {
            Text(name.prefix(1))
                .font(.title.bold())
                .frame(width: 56, height: 56)
                .background(.orange.gradient, in: .circle)
                .foregroundStyle(.white)
            VStack(alignment: .leading) {
                Text(name).font(.headline)
                Text(role).foregroundStyle(.secondary)
            }
            Spacer()
            Button(isFollowing ? "Following" : "Follow", action: onFollow)
                .buttonStyle(.bordered)
        }
        .padding()
    }
}

#Preview {
    ProfileCard(name: "Ada Lovelace", role: "iOS Engineer", isFollowing: false, onFollow: {})
}
```

This shape (**data in, events out as closures**) is the convention for reusable views. You'll write hundreds of them.

## Recap

:::recap
- SwiftUI is **declarative**: describe the UI for the current state; SwiftUI keeps it in sync.
- A view is a lightweight struct with a `body`. Keep `body` fast and side-effect free.
- Modifiers wrap views and return new ones, so **order matters**.
- `@State` holds local state; changing it re-renders the view.
- Compose screens from small views that take data and closures.
:::

## Practice

:::exercise
1. Build a `BusinessCard` view with an SF Symbol, your name (`.title`) and job title (`.secondary`) in a `VStack`, with padding and a rounded background. Add light and dark previews.
2. Extend `CounterView` with a **Reset** button that sets the count to 0, disabled when the count is already 0 (`.disabled(count == 0)`).
:::

:::solution Show solution
```swift
import SwiftUI

struct BusinessCard: View {
    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: "swift")
                .font(.system(size: 40))
                .foregroundStyle(.orange)
            Text("Ada Lovelace").font(.title)
            Text("iOS Developer").foregroundStyle(.secondary)
        }
        .padding(24)
        .background(.background.secondary, in: .rect(cornerRadius: 20))
    }
}

struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 16) {
            Text("You tapped \(count) times").font(.title2)
            HStack {
                Button("Tap me") { count += 1 }
                    .buttonStyle(.borderedProminent)
                Button("Reset") { count = 0 }
                    .buttonStyle(.bordered)
                    .disabled(count == 0)
            }
        }
    }
}

#Preview("Card") { BusinessCard() }
#Preview("Card, dark") { BusinessCard().preferredColorScheme(.dark) }
#Preview("Counter") { CounterView() }
```
:::

## Check your understanding

```quiz
Q: What does "declarative UI" mean in SwiftUI?
- [ ] You write layouts in XML
- [x] You describe the UI for the current state, and SwiftUI updates the screen when state changes
- [ ] You manually update each label
- [ ] UI is designed in Interface Builder only
> UI = f(state).

Q: Why is order important in `.padding().background(.orange)` vs `.background(.orange).padding()`?
- [ ] It isn't
- [x] Each modifier wraps the result of the previous one, so the background covers different areas
- [ ] The second one doesn't compile
- [ ] Only the last modifier is applied
> Modifiers apply from the inside out.

Q: Why is `@State` needed for a counter instead of a plain `var`?
- [ ] @State makes it faster
- [x] SwiftUI stores the value outside the recreated view struct and refreshes the view when it changes
- [ ] Plain properties can't hold Ints
- [ ] @State saves the value to disk
> Views are recreated constantly; @State gives them persistent, observed storage.

Q: What should you never do inside `body`?
- [ ] Use if statements
- [ ] Call other views
- [x] Start network requests or other slow side effects
- [ ] Use modifiers
> body can run many times; side effects belong in actions, `.task` or view models.
```
