---
id: swiftui-controls
title: Text, Images & Controls
part: 3
minutes: 35
summary: A tour of SwiftUI's built-in views, including text and fonts, images and SF Symbols, buttons, text fields, toggles, pickers, sliders and alerts, with Dynamic Type and accessibility in mind.
---

:::goals
- Text styles and **Dynamic Type**
- Images, SF Symbols, `Label` and `AsyncImage`
- Buttons and button styles
- Text input: `TextField`, `SecureField`, `TextEditor`, and the keyboard
- `Toggle`, `Slider`, `Stepper`, `Picker`, `DatePicker`
- Alerts, confirmation dialogs and progress views
:::

## Text

```swift
Text("Recipes")
    .font(.largeTitle)                  // a text style, not a fixed size
    .fontWeight(.bold)
    .foregroundStyle(.primary)
    .multilineTextAlignment(.center)
    .lineLimit(2)

Text("Ready in **35 minutes** with _4_ ingredients")    // Markdown works in string literals
```

### Dynamic Type

iPhone users can choose their preferred text size in Settings, from extra small to huge accessibility sizes, and many do. If you use **text styles** (`.largeTitle`, `.title`, `.headline`, `.body`, `.caption`…), your text scales automatically.

| Style | Typical use |
|---|---|
| `.largeTitle` | Screen titles |
| `.title`, `.title2`, `.title3` | Section and card titles |
| `.headline` | Emphasised row titles |
| `.body` | Normal text (the default) |
| `.callout`, `.subheadline`, `.footnote` | Secondary text |
| `.caption`, `.caption2` | Small labels |

:::danger Avoid fixed font sizes
`.font(.system(size: 17))` ignores the user's text size setting. Prefer text styles. If you truly need a custom size, use `.font(.system(size: 17, design: .rounded))` sparingly, or scale it with `@ScaledMetric`.
:::

Test it: in a preview, use the **Variants → Dynamic Type Variants** control under the canvas to see every size at once.

## Images and SF Symbols

```swift
Image(systemName: "heart.fill")              // SF Symbol
    .font(.title)                            // symbols scale with fonts
    .foregroundStyle(.red)

Image(systemName: "cloud.sun.rain.fill")
    .symbolRenderingMode(.multicolor)        // symbols can be multicolour

Image("pancakes")                            // from Assets.xcassets
    .resizable()                             // allow resizing
    .scaledToFill()                          // fill the frame (may crop)
    .frame(height: 200)
    .clipped()
    .accessibilityLabel("A stack of pancakes with berries")

Label("35 min", systemImage: "clock")        // icon + text, adapts to context
```

Images from the internet load with `AsyncImage`:

```swift
AsyncImage(url: URL(string: "https://cdn.dummyjson.com/recipe-images/1.webp")) { image in
    image.resizable().scaledToFill()
} placeholder: {
    ProgressView()
}
.frame(width: 120, height: 120)
.clipShape(.rect(cornerRadius: 12))
```

:::note Accessibility
VoiceOver reads images aloud using their accessibility label. Add `.accessibilityLabel("…")` to meaningful images, and hide decorative ones with `Image(decorative: "flourish")` or `.accessibilityHidden(true)`. SF Symbols get sensible default labels.
:::

## Buttons

```swift
Button("Save") {
    save()
}

Button("Delete", role: .destructive) { deleteItem() }   // red, and treated specially in menus

Button {
    addToCart()
} label: {
    Label("Add to cart", systemImage: "cart.badge.plus")  // any view as a label
}

// Built-in styles
Button("Primary") { }.buttonStyle(.borderedProminent)
Button("Secondary") { }.buttonStyle(.bordered)
Button("Plain") { }.buttonStyle(.plain)

// Sizing and state
Button("Continue") { }
    .buttonStyle(.borderedProminent)
    .controlSize(.large)
    .disabled(!formIsValid)
```

## Text input

Text fields need a **binding**, `$name`: a two-way connection to some state. The field reads the current value and writes changes back. (Bindings are explained fully in the next lesson.)

```swift title="SignInForm.swift"
import SwiftUI

struct SignInForm: View {
    @State private var email = ""
    @State private var password = ""
    @State private var bio = ""

    var body: some View {
        VStack(spacing: 12) {
            TextField("Email", text: $email)
                .keyboardType(.emailAddress)
                .textContentType(.emailAddress)          // enables AutoFill
                .textInputAutocapitalization(.never)
                .autocorrectionDisabled()
                .textFieldStyle(.roundedBorder)

            SecureField("Password", text: $password)     // hides the text
                .textContentType(.password)
                .textFieldStyle(.roundedBorder)

            TextEditor(text: $bio)                        // multi-line
                .frame(height: 100)
                .border(.gray.opacity(0.3))

            Text("\(email.count) characters typed")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}

#Preview { SignInForm() }
```

Useful keyboard types: `.emailAddress`, `.numberPad`, `.decimalPad`, `.phonePad`, `.URL`. Use `.submitLabel(.done)` and `.onSubmit { }` to handle the Return key.

## Toggles, sliders, steppers and pickers

```swift title="PreferencesView.swift"
import SwiftUI

enum Spice: String, CaseIterable, Identifiable {
    case mild, medium, hot
    var id: Self { self }
}

struct PreferencesView: View {
    @State private var notifications = true
    @State private var servings = 2
    @State private var volume = 0.5
    @State private var spice = Spice.medium
    @State private var date = Date.now

    var body: some View {
        VStack(alignment: .leading, spacing: 20) {
            Toggle("Daily notifications", isOn: $notifications)

            Stepper("Servings: \(servings)", value: $servings, in: 1...12)

            Slider(value: $volume, in: 0...1) {
                Text("Volume")
            }

            Picker("Spice level", selection: $spice) {
                ForEach(Spice.allCases) { level in
                    Text(level.rawValue.capitalized).tag(level)
                }
            }
            .pickerStyle(.segmented)                // also .menu, .wheel, .inline

            DatePicker("Cook on", selection: $date, displayedComponents: .date)
        }
        .padding()
    }
}

#Preview { PreferencesView() }
```

## Progress

```swift
ProgressView()                                   // spinner
ProgressView("Loading recipes…")                 // spinner with a label
ProgressView(value: 0.7)                         // 70% bar
Gauge(value: 0.4) { Text("Battery") }            // gauge
```

## Alerts and confirmation dialogs

Alerts are **declarative** too: you describe *when* they're shown with a Boolean binding.

```swift title="DeleteButton.swift"
import SwiftUI

struct DeleteButton: View {
    @State private var showConfirm = false
    @State private var showDeleted = false

    var body: some View {
        Button("Delete recipe", role: .destructive) {
            showConfirm = true
        }
        .confirmationDialog("Delete this recipe?", isPresented: $showConfirm, titleVisibility: .visible) {
            Button("Delete", role: .destructive) { showDeleted = true }
            Button("Cancel", role: .cancel) { }
        } message: {
            Text("This can't be undone.")
        }
        .alert("Deleted", isPresented: $showDeleted) {
            Button("OK") { }
        } message: {
            Text("The recipe was removed.")
        }
    }
}

#Preview { DeleteButton() }
```

SwiftUI sets `showConfirm` back to `false` automatically when the dialog is dismissed.

## Links and menus

```swift
Link("Swift documentation", destination: URL(string: "https://www.swift.org/documentation/")!)

Menu {
    Button("Sort by name") { }
    Button("Sort by rating") { }
} label: {
    Label("Sort", systemImage: "arrow.up.arrow.down")
}
```

## Recap

:::recap
- Use **text styles** so text respects Dynamic Type.
- SF Symbols via `Image(systemName:)`; `Label` for icon + text; `AsyncImage` for URLs.
- Buttons: title + action, `role`, and styles like `.borderedProminent`.
- Inputs bind to state with `$`: `TextField`, `Toggle`, `Stepper`, `Slider`, `Picker`, `DatePicker`.
- Alerts and dialogs are driven by Boolean state.
:::

## Practice

:::exercise
Build an **Order Coffee** screen:

1. A segmented `Picker` for size (Small, Medium, Large) using an enum.
2. A `Stepper` for sugar (0–5) and a `Toggle` for oat milk.
3. A `TextField` for the customer's name.
4. A summary `Text`: e.g. *"Large coffee for Ada, 2 sugars, oat milk"*.
5. An **Order** button (`.borderedProminent`, `.controlSize(.large)`) disabled while the name is empty, which shows an alert *"Thanks, Ada! Your coffee is on its way."*
:::

:::solution Show solution
```swift
import SwiftUI

enum CoffeeSize: String, CaseIterable, Identifiable {
    case small, medium, large
    var id: Self { self }
}

struct OrderCoffeeView: View {
    @State private var size = CoffeeSize.medium
    @State private var sugar = 0
    @State private var oatMilk = false
    @State private var name = ""
    @State private var showThanks = false

    private var summary: String {
        let who = name.isEmpty ? "you" : name
        return "\(size.rawValue.capitalized) coffee for \(who), \(sugar) sugar\(sugar == 1 ? "" : "s")"
            + (oatMilk ? ", oat milk" : "")
    }

    var body: some View {
        VStack(alignment: .leading, spacing: 20) {
            Picker("Size", selection: $size) {
                ForEach(CoffeeSize.allCases) { Text($0.rawValue.capitalized).tag($0) }
            }
            .pickerStyle(.segmented)

            Stepper("Sugar: \(sugar)", value: $sugar, in: 0...5)
            Toggle("Oat milk", isOn: $oatMilk)
            TextField("Your name", text: $name)
                .textFieldStyle(.roundedBorder)

            Text(summary).foregroundStyle(.secondary)

            Button("Order") { showThanks = true }
                .buttonStyle(.borderedProminent)
                .controlSize(.large)
                .frame(maxWidth: .infinity)
                .disabled(name.trimmingCharacters(in: .whitespaces).isEmpty)
        }
        .padding()
        .alert("Thanks, \(name)!", isPresented: $showThanks) {
            Button("OK") { }
        } message: {
            Text("Your coffee is on its way.")
        }
    }
}

#Preview { OrderCoffeeView() }
```
Notice `summary` is a **computed property**: derived from state, never stored separately.
:::

## Check your understanding

```quiz
Q: Why use `.font(.body)` instead of `.font(.system(size: 17))`?
- [ ] It's shorter
- [x] Text styles scale with the user's Dynamic Type setting
- [ ] System sizes are deprecated
- [ ] .body is bold
> Fixed sizes ignore accessibility text sizes.

Q: What does the `$` in `TextField("Name", text: $name)` do?
- [ ] Formats the text as currency
- [x] Passes a binding, a two-way connection, so the field can read and write the state
- [ ] Makes the field secure
- [ ] Nothing
> Bindings let controls modify state owned elsewhere.

Q: How is an alert shown in SwiftUI?
- [ ] By calling `alert.show()`
- [x] By attaching `.alert(isPresented:)` bound to a Boolean state and setting that state to true
- [ ] By creating a UIAlertController
- [ ] Alerts aren't supported
> Presentation is declarative: state decides whether the alert is visible.

Q: Which view loads and displays an image from a URL?
- [ ] Image(url:)
- [x] AsyncImage
- [ ] URLImage
- [ ] Image(systemName:)
> AsyncImage downloads the image and lets you provide a placeholder.
```
