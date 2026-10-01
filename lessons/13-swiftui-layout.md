---
id: swiftui-layout
title: Layout with Stacks, Frames & Spacing
part: 3
minutes: 35
summary: Arrange views with VStack, HStack and ZStack, control size with frames, spacing and padding, and understand how SwiftUI's layout system actually decides sizes.
---

:::goals
- `VStack`, `HStack`, `ZStack`, and alignment and spacing
- `Spacer`, `Divider`, `padding`, `frame`
- How SwiftUI layout works: parents propose, children choose
- `background` and `overlay`
- Safe areas, `ScrollView` and `ViewThatFits`
- Building a real card layout
:::

## The three stacks

<div class="diagram"><div class="flow">
<div class="hl">VStack<small>children top to bottom ↓</small></div>
<div class="hl">HStack<small>children left to right →</small></div>
<div class="hl">ZStack<small>children layered back to front ⧉</small></div>
</div></div>

```swift title="StacksDemo.swift"
import SwiftUI

struct StacksDemo: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 16) {       // alignment + spacing
            Text("Title").font(.title)
            HStack(spacing: 8) {
                Image(systemName: "star.fill")
                Text("4.8")
                Text("·")
                Text("Italian")
            }
            .foregroundStyle(.secondary)
            ZStack(alignment: .bottomLeading) {
                RoundedRectangle(cornerRadius: 16)
                    .fill(.orange.gradient)
                    .frame(height: 140)
                Text("Caption on top")
                    .foregroundStyle(.white)
                    .padding()
            }
        }
        .padding()
    }
}

#Preview { StacksDemo() }
```

| Stack | `alignment` controls | Options |
|---|---|---|
| `VStack` | horizontal alignment | `.leading`, `.center` (default), `.trailing` |
| `HStack` | vertical alignment | `.top`, `.center`, `.bottom`, `.firstTextBaseline` |
| `ZStack` | both | `.topLeading`, `.center`, `.bottomTrailing`, … |

SwiftUI uses **leading/trailing** instead of left/right so layouts flip correctly for right-to-left languages like Arabic and Hebrew.

## `Spacer` and `Divider`

A `Spacer` expands to take all available space along the stack's axis, which pushes views apart:

```swift
HStack {
    Text("Total")
    Spacer()                 // pushes the price to the far right
    Text("$42.00").bold()
}

VStack {
    Text("Top")
    Spacer()                 // pushes "Bottom" to the bottom of the screen
    Text("Bottom")
}
```

`Divider()` draws a thin line: horizontal in a `VStack`, vertical in an `HStack`.

## Padding and frames

```swift
Text("Hello")
    .padding()                          // system default on all sides
    .padding(.horizontal, 24)           // specific edges
    .padding(.top, 8)

Text("Fixed size")
    .frame(width: 200, height: 50)      // propose an exact size

Text("Full width, leading aligned")
    .frame(maxWidth: .infinity, alignment: .leading)   // take all width available

Image(systemName: "photo")
    .resizable()
    .scaledToFit()
    .frame(maxHeight: 120)
```

## How layout really works

SwiftUI layout is a conversation in three steps:

1. The **parent proposes** a size to its child ("you can have 390 × 700").
2. The **child chooses** its own size ("I only need 80 × 20"). A child can't be forced to be bigger than it wants to be.
3. The **parent positions** the child within its space (using alignment).

That explains a lot:

- `Text` wants just enough space for its text, so a background only covers the text.
- `.frame(maxWidth: .infinity)` doesn't stretch the text. It creates a flexible frame *around* it that takes the full width and aligns the text inside it.
- `Color.orange`, `Spacer()` and `.resizable()` images are flexible: they take whatever they're offered.

## Background and overlay

```swift title="BadgeView.swift"
import SwiftUI

struct BadgeView: View {
    var body: some View {
        Image(systemName: "bell.fill")
            .font(.largeTitle)
            .padding()
            .background(.orange.opacity(0.15), in: .circle)          // behind, sized to the view
            .overlay(alignment: .topTrailing) {                        // in front, sized to the view
                Text("3")
                    .font(.caption.bold())
                    .foregroundStyle(.white)
                    .padding(6)
                    .background(.red, in: .circle)
            }
    }
}

#Preview { BadgeView() }
```

`background` and `overlay` take the **size of the view they modify**, which is often exactly what you want. `ZStack`, by contrast, sizes itself to fit all its children.

## Shapes

```swift
Circle().fill(.blue).frame(width: 40, height: 40)
RoundedRectangle(cornerRadius: 12).stroke(.gray, lineWidth: 1)
Capsule().fill(.green.gradient)
Rectangle().frame(height: 1)

Text("Clipped")
    .padding()
    .background(.yellow)
    .clipShape(.rect(cornerRadius: 10))          // round the corners of any view
```

## Safe areas

iPhones have a notch or Dynamic Island, a home indicator and rounded corners. SwiftUI keeps content inside the **safe area** automatically. Backgrounds often *should* extend to the edges:

```swift
ZStack {
    Color.orange.opacity(0.15)
        .ignoresSafeArea()           // fill the whole screen
    Text("Content stays safe")
}
```

## Scrolling

Stacks don't scroll. When content might not fit (and it often won't with large text sizes), wrap it in a `ScrollView`:

```swift
ScrollView {
    VStack(alignment: .leading, spacing: 12) {
        ForEach(1...50, id: \.self) { i in
            Text("Row \(i)")
        }
    }
    .padding()
}
```

For long, data-driven lists, use `List` or `LazyVStack` (Lesson 16), which only create the rows that are visible.

## Adapting to space

`ViewThatFits` picks the first child that fits the available space. It's great for supporting both small phones and huge accessibility text sizes:

```swift
ViewThatFits {
    HStack { Label("Share", systemImage: "square.and.arrow.up"); Label("Save", systemImage: "heart") }
    VStack { Label("Share", systemImage: "square.and.arrow.up"); Label("Save", systemImage: "heart") }
}
```

## Putting it together: a recipe card

```swift title="RecipeCardView.swift"
import SwiftUI

struct RecipeCardView: View {
    let title: String
    let cuisine: String
    let minutes: Int
    let rating: Double

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            ZStack(alignment: .topTrailing) {
                Rectangle()
                    .fill(.orange.gradient)
                    .frame(height: 140)
                    .overlay {
                        Image(systemName: "fork.knife")
                            .font(.system(size: 44))
                            .foregroundStyle(.white.opacity(0.8))
                    }
                Image(systemName: "heart")
                    .padding(10)
                    .background(.thinMaterial, in: .circle)
                    .padding(10)
            }

            VStack(alignment: .leading, spacing: 6) {
                Text(title)
                    .font(.headline)
                    .lineLimit(1)
                HStack(spacing: 12) {
                    Label("\(minutes) min", systemImage: "clock")
                    Label(rating.formatted(.number.precision(.fractionLength(1))), systemImage: "star.fill")
                    Spacer()
                    Text(cuisine)
                }
                .font(.caption)
                .foregroundStyle(.secondary)
            }
            .padding()
        }
        .background(.background, in: .rect(cornerRadius: 20))
        .clipShape(.rect(cornerRadius: 20))
        .shadow(color: .black.opacity(0.1), radius: 8, y: 4)
    }
}

#Preview {
    RecipeCardView(title: "Classic Margherita Pizza", cuisine: "Italian", minutes: 35, rating: 4.6)
        .padding()
        .background(Color(.systemGroupedBackground))
}
```

`Label` combines an icon and text; `.thinMaterial` gives the frosted-glass blur Apple uses throughout iOS; and `formatted(...)` formats numbers correctly for the user's locale.

## Recap

:::recap
- `VStack`, `HStack`, `ZStack` with `alignment` and `spacing`; `Spacer` pushes views apart.
- Parents propose sizes, children choose, parents position.
- `.frame(maxWidth: .infinity, alignment:)` fills available width; `.padding` adds space.
- `background`/`overlay` match the modified view's size.
- Content stays in the safe area; use `ScrollView` when content might not fit.
:::

## Practice

:::exercise
1. Build a **settings row**: an SF Symbol in a coloured rounded square on the left, a title and subtitle stacked in the middle, and a `chevron.right` on the far right.
2. Build a **stats bar** with three equal-width columns (value on top, label below), separated by vertical `Divider`s. (Hint: give each column `.frame(maxWidth: .infinity)`.)
:::

:::solution Show solution
```swift
import SwiftUI

struct SettingsRow: View {
    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: "bell.badge.fill")
                .foregroundStyle(.white)
                .frame(width: 32, height: 32)
                .background(.red, in: .rect(cornerRadius: 8))
            VStack(alignment: .leading) {
                Text("Notifications").font(.body)
                Text("Daily recipe ideas").font(.caption).foregroundStyle(.secondary)
            }
            Spacer()
            Image(systemName: "chevron.right").foregroundStyle(.tertiary)
        }
        .padding()
    }
}

struct StatsBar: View {
    var body: some View {
        HStack {
            stat("35", "minutes")
            Divider()
            stat("4", "servings")
            Divider()
            stat("300", "kcal")
        }
        .frame(height: 60)
        .padding(.vertical, 8)
        .background(.background.secondary, in: .rect(cornerRadius: 16))
    }

    private func stat(_ value: String, _ label: String) -> some View {
        VStack {
            Text(value).font(.title3.bold())
            Text(label).font(.caption).foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
    }
}

#Preview {
    VStack { SettingsRow(); StatsBar() }.padding()
}
```
Notice `stat(_:_:)`: a small helper function returning `some View`. It's a lightweight alternative to a separate struct for pieces used only inside one view.
:::

## Check your understanding

```quiz
Q: You want an icon on the left and a price pushed to the far right of a row. What do you put between them in an HStack?
- [ ] Divider()
- [x] Spacer()
- [ ] .padding()
- [ ] ZStack
> Spacer expands to fill the available space along the stack's axis.

Q: Why does `.background(.red)` on a Text only colour the text's area?
- [ ] A bug
- [x] Text only chooses the size it needs, and the background takes the size of the view it modifies
- [ ] Backgrounds always fill the screen
- [ ] You need a ZStack
> Parents propose, children choose. Text wants only its own size.

Q: How do you make a view take all available width with its content aligned to the leading edge?
- [x] `.frame(maxWidth: .infinity, alignment: .leading)`
- [ ] `.frame(width: .infinity)`
- [ ] `.padding(.leading)`
- [ ] `.alignment(.leading)`
> A flexible frame fills the width; its alignment positions the content inside.

Q: Why does SwiftUI use `.leading` instead of `.left`?
- [ ] It's shorter
- [x] So layouts automatically mirror for right-to-left languages
- [ ] .left is deprecated in iOS 27
- [ ] No reason
> Leading is the start of the reading direction.
```
