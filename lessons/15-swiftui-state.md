---
id: swiftui-state
title: State, Bindings & Data Flow
part: 3
minutes: 35
summary: The heart of SwiftUI. Learn who owns each piece of data, how @State and @Binding connect views, the single source of truth, and how to react to changes.
---

:::goals
- `@State` for local, view-owned values
- `@Binding` to let a child view read *and write* its parent's state
- The **single source of truth** and data flowing down, events flowing up
- Derived values instead of duplicated state
- `onChange`, `@FocusState` and keeping state in the right place
:::

## Every piece of data has one owner

SwiftUI's golden rule: **each piece of state has exactly one source of truth**. Other views either receive a read-only copy (a plain property) or a **binding** to it. Never keep two copies of the same value in sync by hand, because that's where bugs come from.

| You need | Use |
|---|---|
| A value this view owns and changes (toggle, text, selection) | `@State private var` |
| To read a value from the parent | a plain `let` property |
| To read **and write** a value owned by the parent | `@Binding var` |
| A shared model object (view model, store) | `@Observable` class (Lesson 21) |
| App-wide values like colour scheme or locale | `@Environment` |

## `@State`

`@State` gives a view its own storage that survives view updates and triggers a refresh when it changes. Mark it `private`, because it belongs to this view alone.

```swift title="StateView.swift"
import SwiftUI

struct StateView: View {
    @State private var name = ""
    @State private var isExpanded = false

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            TextField("Your name", text: $name)          // $name = a binding to the state
                .textFieldStyle(.roundedBorder)

            Text(name.isEmpty ? "Hello, stranger!" : "Hello, \(name)!")

            Button(isExpanded ? "Show less" : "Show more") {
                isExpanded.toggle()
            }
            if isExpanded {
                Text("Here are some extra details that only appear when expanded.")
            }
        }
        .padding()
    }
}

#Preview { StateView() }
```

Writing `$name` gives a **binding**: a reference that lets another view (here, the `TextField`) read and write the state.

## `@Binding`: sharing state with a child

When a child view needs to change state owned by its parent, the parent passes a binding, and the child declares it with `@Binding`:

```swift title="WaterTracker.swift"
import SwiftUI

struct CounterRow: View {
    let title: String
    @Binding var count: Int                 // not owned here: read/write access to the parent's value

    var body: some View {
        HStack {
            Text("\(title): \(count)")
            Spacer()
            Button { count -= 1 } label: { Image(systemName: "minus.circle") }
                .disabled(count == 0)
            Button { count += 1 } label: { Image(systemName: "plus.circle") }
        }
        .font(.title3)
    }
}

struct WaterTracker: View {
    @State private var glasses = 0          // ← the single source of truth

    var body: some View {
        VStack(spacing: 20) {
            CounterRow(title: "Glasses of water", count: $glasses)   // pass a binding down
            ProgressView(value: Double(glasses), total: 8)
            if glasses >= 8 {
                Text("🎉 Daily goal reached!")
            }
            Button("Reset") { glasses = 0 }
        }
        .padding()
    }
}

#Preview { WaterTracker() }
```

`CounterRow` doesn't own the count. It only has access to it. When it changes `count`, it's really changing `WaterTracker.glasses`, and **both** views update.

<div class="diagram"><div class="flow">
<div class="hl">WaterTracker<small>@State glasses (the owner)</small></div>
<div class="arrow">⇄</div>
<div>CounterRow<small>@Binding count</small></div>
<div>ProgressView<small>reads glasses</small></div>
</div><div class="diagram-caption">Data flows down; changes flow back up through the binding to the single owner.</div></div>

### Previewing views with bindings

Use `@Previewable` to create state right inside a preview:

```swift
#Preview {
    @Previewable @State var count = 3
    CounterRow(title: "Coffees", count: $count)
        .padding()
}
```

For a fixed value you can also pass `.constant(3)`, a binding that never changes.

## Closures: the other way to send events up

Bindings are great for simple values. For actions, pass a **closure**:

```swift
struct DeleteRow: View {
    let title: String
    let onDelete: () -> Void               // the parent decides what "delete" means

    var body: some View {
        HStack {
            Text(title)
            Spacer()
            Button("Delete", role: .destructive, action: onDelete)
        }
    }
}
```

## Derive, don't duplicate

If a value can be computed from other state, **compute it**, and don't store it:

```swift title="PasswordField.swift"
import SwiftUI

struct PasswordField: View {
    @State private var password = ""

    // Derived values: always correct, never out of sync
    private var hasDigit: Bool { password.contains(where: \.isNumber) }
    private var isLongEnough: Bool { password.count >= 8 }
    private var isStrong: Bool { hasDigit && isLongEnough }

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            SecureField("Password", text: $password)
                .textFieldStyle(.roundedBorder)
            Label("At least 8 characters", systemImage: isLongEnough ? "checkmark.circle.fill" : "circle")
            Label("Contains a number", systemImage: hasDigit ? "checkmark.circle.fill" : "circle")
            Text(isStrong ? "Strong password 💪" : "Keep going…")
                .foregroundStyle(isStrong ? .green : .secondary)
        }
        .padding()
    }
}

#Preview { PasswordField() }
```

## Reacting to changes: `onChange`

Run code when a value changes (for side effects, not for deriving values):

```swift
.onChange(of: searchText) { oldValue, newValue in
    print("Search changed from \(oldValue) to \(newValue)")
}
```

## Focus: `@FocusState`

Control which text field has the keyboard:

```swift title="LoginFields.swift"
import SwiftUI

struct LoginFields: View {
    enum Field { case email, password }
    @State private var email = ""
    @State private var password = ""
    @FocusState private var focused: Field?

    var body: some View {
        VStack {
            TextField("Email", text: $email)
                .focused($focused, equals: .email)
                .submitLabel(.next)
                .onSubmit { focused = .password }     // Return moves to the next field
            SecureField("Password", text: $password)
                .focused($focused, equals: .password)
                .submitLabel(.go)
                .onSubmit { focused = nil }            // dismiss the keyboard
        }
        .textFieldStyle(.roundedBorder)
        .padding()
        .onAppear { focused = .email }
    }
}

#Preview { LoginFields() }
```

## State and identity

SwiftUI keeps `@State` alive as long as the view keeps its **identity**, meaning it stays in the same position in the view tree. If an `if/else` switches between two different views, each branch has its own identity and its own state. This is also why lists need stable IDs (next lesson).

:::warning Don't initialise @State from a parameter and expect updates
`@State private var count: Int` initialised from a parent's value only uses that value **once**, when the view first appears. If the parent's value changes later, the state won't follow. If the parent owns the value, use a plain `let` (read only) or `@Binding` (read/write) instead.
:::

## Recap

:::recap
- One **source of truth** per value.
- `@State private var` = owned by this view. `$state` = a binding to it.
- `@Binding var` = read/write access to a parent's state. Closures send actions up.
- Compute derived values; don't store them.
- `onChange` for side effects, `@FocusState` for the keyboard.
:::

## Practice

:::exercise
Build a **tip calculator**:

1. A `TextField` for the bill amount (use `TextField("Amount", value: $amount, format: .number)` with a `Double` state), a segmented `Picker` for 10/15/20%, and a `Toggle` "Round up".
2. Show the tip and total using **derived** computed properties, formatted as currency with `.formatted(.currency(code: "USD"))`.
3. Extract the percentage picker into its own `TipPicker` view that takes `@Binding var percent: Int`.
:::

:::solution Show solution
```swift
import SwiftUI

struct TipPicker: View {
    @Binding var percent: Int

    var body: some View {
        Picker("Tip", selection: $percent) {
            ForEach([10, 15, 20], id: \.self) { Text("\($0)%").tag($0) }
        }
        .pickerStyle(.segmented)
    }
}

struct TipCalculator: View {
    @State private var amount = 0.0
    @State private var percent = 15
    @State private var roundUp = false

    private var tip: Double {
        let raw = amount * Double(percent) / 100
        return roundUp ? raw.rounded(.up) : raw
    }
    private var total: Double { amount + tip }

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            TextField("Amount", value: $amount, format: .number)
                .keyboardType(.decimalPad)
                .textFieldStyle(.roundedBorder)
            TipPicker(percent: $percent)
            Toggle("Round up tip", isOn: $roundUp)
            Text("Tip: \(tip.formatted(.currency(code: "USD")))").font(.title2)
            Text("Total: \(total.formatted(.currency(code: "USD")))").font(.headline)
        }
        .padding()
    }
}

#Preview { TipCalculator() }
```
:::

## Check your understanding

```quiz
Q: A child view needs to change a Boolean owned by its parent. What should the child declare?
- [ ] `@State var isOn: Bool`
- [x] `@Binding var isOn: Bool`
- [ ] `let isOn: Bool`
- [ ] `var isOn: Bool`
> A binding gives read/write access to state owned elsewhere.

Q: What does `$name` produce when `name` is `@State`?
- [ ] A copy of the string
- [x] A Binding<String> connected to the state
- [ ] The string formatted as currency
- [ ] An optional
> The $ prefix accesses the property wrapper's projected value, a binding.

Q: You have `@State var items` and want to show "3 items left". What's best?
- [ ] Another @State var remainingCount that you update everywhere
- [x] A computed property derived from items
- [ ] A global variable
- [ ] onChange to copy the count
> Derived values can never get out of sync with their source.

Q: Why should `@State` properties be `private`?
- [ ] For performance
- [x] Because they're owned by that view alone; outside views should receive data or bindings instead
- [ ] Swift requires it
- [ ] So they're saved to disk
> Keeping state private makes ownership clear.
```
