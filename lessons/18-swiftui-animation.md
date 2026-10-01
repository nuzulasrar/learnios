---
id: swiftui-animation
title: Animation & Transitions
part: 3
minutes: 25
summary: SwiftUI makes animation almost effortless. Learn implicit and explicit animations, springs, transitions, symbol effects, matched geometry and phase animations.
---

:::goals
- Explicit animations with `withAnimation`
- Implicit animations with `.animation(_:value:)`
- Springs, timing curves and durations
- Transitions for views appearing and disappearing
- `symbolEffect`, `contentTransition`, `matchedGeometryEffect` and `phaseAnimator`
:::

## Animation is a state change

In SwiftUI you don't animate views directly. You **change state** and tell SwiftUI to animate the difference between the old and new UI.

### Explicit: `withAnimation`

Wrap the state change:

```swift title="ExpandingCard.swift"
import SwiftUI

struct ExpandingCard: View {
    @State private var expanded = false

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            HStack {
                Text("Tips").font(.headline)
                Spacer()
                Image(systemName: "chevron.down")
                    .rotationEffect(.degrees(expanded ? 180 : 0))
            }
            if expanded {
                Text("Salt your pasta water generously, and save a cup before draining.")
                    .transition(.opacity.combined(with: .move(edge: .top)))
            }
        }
        .padding()
        .background(.background.secondary, in: .rect(cornerRadius: 16))
        .onTapGesture {
            withAnimation(.spring(duration: 0.4, bounce: 0.3)) {
                expanded.toggle()
            }
        }
        .padding()
    }
}

#Preview { ExpandingCard() }
```

Everything that changes because of `expanded` animates: the rotation, the text appearing, and the card growing.

### Implicit: `.animation(_:value:)`

Attach an animation to a view; it animates whenever `value` changes:

```swift title="FavoriteButton.swift"
import SwiftUI

struct FavoriteButton: View {
    @State private var isFavorite = false

    var body: some View {
        Button {
            isFavorite.toggle()
        } label: {
            Image(systemName: isFavorite ? "heart.fill" : "heart")
                .font(.largeTitle)
                .foregroundStyle(isFavorite ? .red : .secondary)
                .scaleEffect(isFavorite ? 1.2 : 1)
                .animation(.spring(bounce: 0.5), value: isFavorite)
        }
        .accessibilityLabel(isFavorite ? "Remove from favourites" : "Add to favourites")
    }
}

#Preview { FavoriteButton() }
```

:::tip Always pass a value
`.animation(.default, value: x)` only animates changes caused by `x`. The old form without `value:` was deprecated because it animated *everything*, often by surprise.
:::

## Kinds of animation

| Animation | Feel |
|---|---|
| `.default` | A sensible spring |
| `.spring(duration:bounce:)` | Physics-based and natural, the Apple default |
| `.snappy`, `.smooth`, `.bouncy` | Preset springs |
| `.easeInOut(duration: 0.3)` | Classic curve with a fixed duration |
| `.linear(duration: 1)` | Constant speed (progress, rotation) |
| `.default.repeatForever(autoreverses: true)` | Loops, e.g. for a pulsing effect |

Keep UI animations short, around 0.2 to 0.5 s. They should explain what changed, not slow people down.

## Transitions

A **transition** controls how a view is **inserted or removed** (inside an `if` or a `ForEach`):

```swift
if showBanner {
    BannerView()
        .transition(.move(edge: .top).combined(with: .opacity))
}
// Others: .opacity, .scale, .slide, .push(from: .trailing), .asymmetric(insertion:removal:)
```

Transitions only run if the change is animated (`withAnimation` or `.animation(_:value:)` on a container).

## Content transitions

Animate changes to text and symbols themselves:

```swift title="LiveCounter.swift"
import SwiftUI

struct LiveCounter: View {
    @State private var count = 0
    @State private var isMuted = false

    var body: some View {
        VStack(spacing: 24) {
            Text("\(count)")
                .font(.system(size: 64, weight: .bold, design: .rounded))
                .contentTransition(.numericText(value: Double(count)))   // digits roll

            Image(systemName: isMuted ? "speaker.slash.fill" : "speaker.wave.3.fill")
                .font(.largeTitle)
                .contentTransition(.symbolEffect(.replace))              // symbol morphs

            HStack {
                Button("Add") { withAnimation { count += 1 } }
                Button("Mute") { withAnimation { isMuted.toggle() } }
            }
            .buttonStyle(.bordered)
        }
    }
}

#Preview { LiveCounter() }
```

## Symbol effects

SF Symbols have built-in animations:

```swift
Image(systemName: "bell.fill")
    .symbolEffect(.bounce, value: notificationCount)   // bounces when the value changes

Image(systemName: "wifi")
    .symbolEffect(.variableColor.iterative, isActive: isSearching)   // loops while active
```

## Matched geometry: smooth hero animations

`matchedGeometryEffect` animates a view "moving" between two places in your layout:

```swift title="TabHighlight.swift"
import SwiftUI

struct TabHighlight: View {
    @Namespace private var namespace
    @State private var selected = "Breakfast"
    let tabs = ["Breakfast", "Lunch", "Dinner"]

    var body: some View {
        HStack {
            ForEach(tabs, id: \.self) { tab in
                Text(tab)
                    .padding(.vertical, 8)
                    .padding(.horizontal, 14)
                    .background {
                        if selected == tab {
                            Capsule()
                                .fill(.orange.opacity(0.2))
                                .matchedGeometryEffect(id: "highlight", in: namespace)
                        }
                    }
                    .onTapGesture {
                        withAnimation(.snappy) { selected = tab }
                    }
            }
        }
    }
}

#Preview { TabHighlight() }
```

For zooming from a thumbnail into a detail screen, `NavigationLink` supports `.navigationTransition(.zoom(sourceID:in:))`.

## Multi-step animations: `phaseAnimator`

```swift
Image(systemName: "hand.wave.fill")
    .font(.largeTitle)
    .phaseAnimator([0, -20, 20, 0]) { view, angle in
        view.rotationEffect(.degrees(angle))
    }
```

The view cycles through the phases automatically, which is great for attention-grabbing effects used sparingly.

## Respect Reduce Motion

Some users turn on **Reduce Motion** because large animations can cause discomfort. Check it and tone things down:

```swift
@Environment(\.accessibilityReduceMotion) private var reduceMotion

withAnimation(reduceMotion ? nil : .spring) { expanded.toggle() }
```

## Recap

:::recap
- Animate by changing state inside `withAnimation`, or with `.animation(_:value:)`.
- Springs are the natural default; keep animations short.
- `.transition` for insertion and removal; `.contentTransition` for changing text and symbols.
- `symbolEffect`, `matchedGeometryEffect` and `phaseAnimator` for extra polish.
- Respect Reduce Motion.
:::

:::exercise
1. Build a **like button** with a heart that pops (scale spring), turns red, and shows a count that rolls with `.numericText()`.
2. Build a **toast**: a capsule message that slides down from the top when a button is tapped and disappears after 2 seconds. (Hint: use `.task(id:)` or `Task { try? await Task.sleep(for: .seconds(2)); withAnimation { show = false } }`.)
:::

:::solution Show solution for task 2
```swift
import SwiftUI

struct ToastDemo: View {
    @State private var showToast = false

    var body: some View {
        ZStack(alignment: .top) {
            Button("Save recipe") {
                withAnimation(.snappy) { showToast = true }
            }
            .buttonStyle(.borderedProminent)
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            if showToast {
                Label("Saved to favourites", systemImage: "heart.fill")
                    .padding()
                    .background(.regularMaterial, in: .capsule)
                    .transition(.move(edge: .top).combined(with: .opacity))
                    .task {
                        try? await Task.sleep(for: .seconds(2))
                        withAnimation { showToast = false }
                    }
            }
        }
    }
}

#Preview { ToastDemo() }
```
`.task` on the toast starts when it appears and is cancelled automatically if it disappears early.
:::

## Check your understanding

```quiz
Q: How do you animate a change in SwiftUI?
- [ ] Call view.animate()
- [x] Change state inside withAnimation, or attach .animation(_:value:) to the view
- [ ] Use a timer to update the frame manually
- [ ] Animations require UIKit
> Animations describe how to move between two states.

Q: What does a transition control?
- [ ] The colour of a view
- [x] How a view animates when it's inserted into or removed from the hierarchy
- [ ] Navigation between screens only
- [ ] The duration of all animations
> Transitions apply to views appearing and disappearing.

Q: Why pass `value:` to `.animation`?
- [ ] It sets the duration
- [x] So only changes to that value are animated, avoiding surprising animations
- [ ] It's optional and has no effect
- [ ] To repeat the animation
> Scoping animations to a value keeps them predictable.
```
