---
id: swiftui-styling
title: Styling, Colours, Dark Mode & Liquid Glass
part: 3
minutes: 30
summary: Make your app look polished and at home on iOS. Semantic colours and dark mode, asset catalog colours, fonts, SF Symbols rendering, reusable styles, materials and Apple's Liquid Glass design.
---

:::goals
- Semantic colours, and why they make dark mode free
- Custom colours in the asset catalog; the accent colour
- Font design, weights and custom fonts
- SF Symbols variants and rendering modes
- Reusable styling with `ViewModifier` and `ButtonStyle`
- Materials, gradients and Liquid Glass
- App icon
:::

## Semantic colours

iOS provides **semantic** colours that adapt automatically to light mode, dark mode, and high-contrast settings:

```swift
Text("Title").foregroundStyle(.primary)          // black in light, white in dark
Text("Subtitle").foregroundStyle(.secondary)     // softer grey that adapts
Text("Hint").foregroundStyle(.tertiary)
Rectangle().fill(.background)                    // the screen background
Rectangle().fill(.background.secondary)          // a grouped / card background
Button("Go") { }.tint(.orange)                   // tint for controls
```

:::danger Hard-coded black and white break dark mode
`Color.black` text on a dark background is invisible. Use `.primary`, `.secondary` and `.background` styles, and your app supports dark mode with almost no extra work. Test with the dark-mode control in the canvas or [[Cmd]] + [[Shift]] + [[A]] in the Simulator.
:::

## Custom colours

Define brand colours in **Assets.xcassets**: click **+ → Color Set**, name it (e.g. `Brand`), and set separate **Any** (light) and **Dark** appearances. Use it with the generated symbol:

```swift
Text("RecipeBox").foregroundStyle(Color(.brand))     // generated from the asset name
// or: Color("Brand")
```

The **AccentColor** asset sets your app's tint for buttons, links, toggles and selections everywhere. It's one change for a consistent brand.

In code you can also use the built-in palette (`.orange`, `.mint`, `.indigo`…), which is carefully tuned for both modes, and `.gradient` for a subtle gradient of any colour: `.fill(.orange.gradient)`.

## Fonts

```swift
Text("Hello").font(.title.weight(.semibold))
Text("Hello").font(.system(.largeTitle, design: .rounded, weight: .bold))   // rounded design
Text("12:45").font(.body.monospacedDigit())                                  // digits that don't jiggle
Text("Code").font(.system(.body, design: .monospaced))
Text("Elegant").font(.system(.title, design: .serif))
```

To add a custom font: drag the `.ttf`/`.otf` into your project (tick your target), add it to the **Info** tab under *Fonts provided by application*, then use `.font(.custom("Nunito-Bold", size: 20, relativeTo: .title))`. The `relativeTo:` parameter keeps it scaling with Dynamic Type.

## SF Symbols

```swift
Image(systemName: "heart.fill").foregroundStyle(.red)

Image(systemName: "cloud.sun.rain.fill")
    .symbolRenderingMode(.multicolor)              // Apple's built-in colours

Image(systemName: "person.3.sequence.fill")
    .symbolRenderingMode(.hierarchical)            // one colour at different opacities
    .foregroundStyle(.orange)

Image(systemName: "wifi", variableValue: 0.5)      // variable symbols show levels

Image(systemName: "star")
    .symbolVariant(.fill)                          // same as "star.fill"
    .imageScale(.large)
```

Browse and search symbols in the free **SF Symbols** app. Check each symbol's availability, since newer symbols need newer iOS versions.

## Reusable styles

When the same combination of modifiers appears in many places, wrap it in a **`ViewModifier`**:

```swift title="CardStyle.swift"
import SwiftUI

struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .frame(maxWidth: .infinity, alignment: .leading)
            .background(.background.secondary, in: .rect(cornerRadius: 16))
    }
}

extension View {
    func cardStyle() -> some View { modifier(CardStyle()) }   // nicer call site
}

struct CardDemo: View {
    var body: some View {
        VStack(spacing: 12) {
            Text("Ingredients").font(.headline).cardStyle()
            Text("Steps").font(.headline).cardStyle()
        }
        .padding()
    }
}

#Preview { CardDemo() }
```

Custom **button styles** let you change how every button looks and reacts to presses:

```swift title="PillButtonStyle.swift"
import SwiftUI

struct PillButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .font(.headline)
            .padding(.horizontal, 24)
            .padding(.vertical, 12)
            .background(.orange.gradient, in: .capsule)
            .foregroundStyle(.white)
            .scaleEffect(configuration.isPressed ? 0.95 : 1)       // press feedback
            .animation(.snappy, value: configuration.isPressed)
    }
}

extension ButtonStyle where Self == PillButtonStyle {
    static var pill: PillButtonStyle { PillButtonStyle() }
}

#Preview {
    Button("Start cooking") { }.buttonStyle(.pill)
}
```

## Materials

**Materials** are translucent, blurred backgrounds that let content behind them show through subtly. They look great over images and adapt to light and dark:

```swift
Text("Over a photo")
    .padding()
    .background(.ultraThinMaterial, in: .rect(cornerRadius: 12))
// .ultraThinMaterial, .thinMaterial, .regularMaterial, .thickMaterial
```

## Liquid Glass

Since iOS 26, Apple's design language is **Liquid Glass**: a translucent, light-refracting material used for navigation bars, tab bars, toolbars, sheets and controls. **Standard SwiftUI components adopt it automatically** when you build with the current SDK: `NavigationStack`, `TabView`, toolbars and system buttons all get the new look for free.

For your own floating controls, apply it with `glassEffect`:

```swift title="GlassBadge.swift"
import SwiftUI

struct GlassBadge: View {
    var body: some View {
        ZStack {
            LinearGradient(colors: [.orange, .pink], startPoint: .topLeading, endPoint: .bottomTrailing)
                .ignoresSafeArea()
            Label("4.8 · 1.2k reviews", systemImage: "star.fill")
                .font(.headline)
                .padding()
                .glassEffect()                          // the Liquid Glass material
        }
    }
}

#Preview { GlassBadge() }
```

There are also glass button styles, `.buttonStyle(.glass)` and `.buttonStyle(.glassProminent)`, for buttons that float over content.

:::tip Use glass sparingly
Liquid Glass is meant for the **navigation layer** that floats above your content (bars, controls, floating buttons), not for every card and row. Let the system components handle most of it, and keep your content itself clear and readable.
:::

If you need to support iOS versions before 26, wrap it: `if #available(iOS 26, *) { … } else { … }`.

## Dark mode and previews

```swift
#Preview("Light") { CardDemo() }
#Preview("Dark") { CardDemo().preferredColorScheme(.dark) }
```

Read the current scheme in a view with `@Environment(\.colorScheme) private var colorScheme`, but prefer adaptive colours so you rarely need it.

## App icon

In `Assets.xcassets`, select **AppIcon** and drop in a **1024×1024** image. You can supply separate **Dark** and **Tinted** variants for iOS's icon appearance options. For the full layered Liquid Glass look, Apple provides the **Icon Composer** app to build icons from layers.

## Recap

:::recap
- Use semantic styles (`.primary`, `.secondary`, `.background`) and asset colours with light/dark variants.
- AccentColor sets your brand tint app-wide.
- Font design and weight via `.font(.system(_, design:, weight:))`; custom fonts with `relativeTo:`.
- Reuse styling with `ViewModifier` and `ButtonStyle`.
- Materials and Liquid Glass (`.glassEffect()`) for the floating navigation layer.
:::

:::exercise
1. Create a `Brand` colour set with light and dark values, and set **AccentColor** to match.
2. Write a `ViewModifier` called `TagStyle` for small capsule-shaped tags (caption font, horizontal padding, tinted background at 15% opacity) and use it on three cuisine tags in an `HStack`.
3. Preview your screen in light and dark mode and at the largest Dynamic Type size, and fix anything that breaks.
:::

## Check your understanding

```quiz
Q: What's the easiest way to make text readable in both light and dark mode?
- [ ] Always use Color.black
- [x] Use semantic styles like .primary and .secondary
- [ ] Check the colour scheme in every view
- [ ] Disable dark mode
> Semantic colours adapt automatically.

Q: What does the AccentColor asset control?
- [ ] The app icon
- [x] The default tint for buttons, links, toggles and selection across the app
- [ ] The launch screen
- [ ] Only navigation titles
> Set it once for a consistent brand colour.

Q: When should you create a custom ViewModifier?
- [ ] For every view
- [x] When the same combination of modifiers is repeated in many places
- [ ] Only for text
- [ ] Never. Use copy and paste
> ViewModifiers make styling reusable and consistent.

Q: How do standard components like NavigationStack and TabView get the Liquid Glass look?
- [ ] You must apply .glassEffect() to each one
- [x] Automatically, when built with the current SDK
- [ ] Only with a special library
- [ ] Only on iPad
> System components adopt Liquid Glass for free; use glassEffect for your own floating controls.
```
