---
id: observation-mvvm
title: "Observation & MVVM: View Models"
part: 4
minutes: 40
summary: Move logic and state out of views into @Observable models. Model loading states, share models with @Environment, and use @Bindable, the modern way to structure SwiftUI screens.
---

:::goals
- The `@Observable` macro and how SwiftUI tracks what you read
- Owning a model with `@State`, passing it down, editing it with `@Bindable`
- Sharing models app-wide with `@Environment`
- The MVVM pattern: views, view models, models
- Loading / loaded / failed states with an enum
- Making view models `@MainActor`
:::

## Why move logic out of views?

So far, state has lived in views with `@State`. That's perfect for simple UI state, like whether a section is expanded. But as screens grow:

- Views fill up with logic (validation, loading, filtering) that's hard to test.
- Several views need to share the same data.
- Previews and tests need a way to substitute sample data.

The answer is an **observable model class**.

## `@Observable`

Mark a class with the `@Observable` macro and SwiftUI automatically tracks which of its properties each view **reads**. When one of those properties changes, only the views that read it update.

```swift title="CounterModel.swift"
import SwiftUI
import Observation

@Observable
final class CounterModel {
    var count = 0
    var step = 1

    func increment() { count += step }
    func reset() { count = 0 }
}

struct ObservedCounterView: View {
    @State private var model = CounterModel()      // this view OWNS the model

    var body: some View {
        VStack(spacing: 16) {
            Text("\(model.count)").font(.system(size: 60, weight: .bold, design: .rounded))
            Stepper("Step: \(model.step)", value: $model.step, in: 1...10)
            HStack {
                Button("Add", action: model.increment).buttonStyle(.borderedProminent)
                Button("Reset", action: model.reset).buttonStyle(.bordered)
            }
        }
        .padding()
    }
}

#Preview { ObservedCounterView() }
```

- **Own** a model with `@State` so it's created once and kept alive for the view's lifetime.
- `$model.step` gives a binding to a property of the model.
- No special property wrappers are needed inside the class: just `var`s.

:::note Older code: ObservableObject
Before iOS 17, the equivalent was `class X: ObservableObject` with `@Published` properties and `@StateObject` / `@ObservedObject` / `@EnvironmentObject` in views. You'll see it in older tutorials and codebases. `@Observable` is simpler and more efficient; use it for new code.
:::

## Passing models to child views

A child that only **reads** a model just takes it as a normal property. SwiftUI still tracks what it reads:

```swift
struct CountBadge: View {
    let model: CounterModel          // no wrapper needed to read and observe
    var body: some View { Text("Count: \(model.count)") }
}
```

A child that needs **bindings** to the model's properties uses **`@Bindable`**:

```swift
struct StepEditor: View {
    @Bindable var model: CounterModel
    var body: some View {
        Stepper("Step: \(model.step)", value: $model.step)   // $ works thanks to @Bindable
    }
}
```

## Sharing app-wide with `@Environment`

For models many screens need (the signed-in user, a shopping cart, settings), put them in the **environment** once, and any view can read them:

```swift title="CartExample.swift"
import SwiftUI

@Observable
final class Cart {
    var items: [String] = []
    var count: Int { items.count }
    func add(_ item: String) { items.append(item) }
}

struct ShopView: View {
    @Environment(Cart.self) private var cart            // read it from the environment

    var body: some View {
        VStack(spacing: 12) {
            ForEach(["Flour", "Eggs", "Milk"], id: \.self) { item in
                Button("Add \(item)") { cart.add(item) }
            }
            CartBadge()
        }
    }
}

struct CartBadge: View {
    @Environment(Cart.self) private var cart
    var body: some View { Label("\(cart.count) items", systemImage: "cart") }
}

#Preview {
    ShopView()
        .environment(Cart())                             // provide it at the top
}
```

In the app you'd create it once in your `App` struct: `@State private var cart = Cart()` and `.environment(cart)` on the `WindowGroup`'s root view.

:::warning Missing environment objects crash
If a view reads `@Environment(Cart.self)` but no ancestor provided one with `.environment(...)`, the app crashes. Remember to provide it in previews too.
:::

## MVVM in SwiftUI

**MVVM** (Model–View–ViewModel) is the most common architecture for SwiftUI screens:

<div class="diagram"><div class="flow">
<div>Model<small>plain data: structs like Recipe</small></div>
<div class="arrow">←</div>
<div class="hl">ViewModel<small>@Observable class: state + logic</small></div>
<div class="arrow">⇄</div>
<div>View<small>SwiftUI struct: displays state, forwards actions</small></div>
</div></div>

- The **View** displays the view model's state and calls its methods on user actions. It contains no business logic.
- The **ViewModel** holds the screen's state, loads data (via services, next lesson), validates input and exposes simple properties for the view.
- The **Model** is your plain data.

## Loading states

Model "loading → success or failure" with an enum, so impossible combinations (loading *and* failed at once) can't happen:

```swift title="QuotesScreen.swift"
import SwiftUI

struct Quote: Identifiable {
    let id: Int
    let text: String
    let author: String
}

enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case failed(String)
}

@MainActor
@Observable
final class QuotesViewModel {
    private(set) var state: LoadState<[Quote]> = .idle      // views can read, only the VM can write

    func load() async {
        state = .loading
        do {
            try await Task.sleep(for: .seconds(1))          // pretend network call
            if Bool.random() { throw URLError(.notConnectedToInternet) }
            state = .loaded([
                Quote(id: 1, text: "Simplicity is prerequisite for reliability.", author: "Edsger Dijkstra"),
                Quote(id: 2, text: "Make it work, make it right, make it fast.", author: "Kent Beck")
            ])
        } catch {
            state = .failed("Couldn't load quotes. Check your connection.")
        }
    }
}

struct QuotesScreen: View {
    @State private var viewModel = QuotesViewModel()

    var body: some View {
        NavigationStack {
            Group {
                switch viewModel.state {
                case .idle, .loading:
                    ProgressView("Loading…")
                case .loaded(let quotes):
                    List(quotes) { quote in
                        VStack(alignment: .leading, spacing: 4) {
                            Text("“\(quote.text)”")
                            Text(quote.author).font(.caption).foregroundStyle(.secondary)
                        }
                    }
                    .refreshable { await viewModel.load() }
                case .failed(let message):
                    ContentUnavailableView {
                        Label("Offline", systemImage: "wifi.slash")
                    } description: {
                        Text(message)
                    } actions: {
                        Button("Try again") { Task { await viewModel.load() } }
                    }
                }
            }
            .navigationTitle("Quotes")
            .task { await viewModel.load() }                   // load when the screen appears
        }
    }
}

#Preview { QuotesScreen() }
```

Note these details:

- **`@MainActor`** on the view model: its state drives the UI, so it lives on the main actor. `await` calls inside still don't block.
- **`private(set)`**: the view can read `state` but can't change it. All changes go through the view model's methods.
- `.task` loads once when the view appears; `.refreshable` enables pull-to-refresh.

:::tip The screen stays simple
Look at `QuotesScreen`: it contains **no logic**, just a `switch` over states. That makes the UI easy to reason about, and the view model easy to test (Lesson 31).
:::

## Recap

:::recap
- `@Observable` classes: SwiftUI tracks exactly the properties each view reads.
- Own with `@State`, pass as a plain property, edit with `@Bindable`, share with `.environment()` / `@Environment(Type.self)`.
- MVVM: views display and forward actions; view models hold state and logic.
- Model loading with an enum; mark view models `@MainActor`; expose state with `private(set)`.
:::

## Practice

:::exercise
Refactor the **tip calculator** from Lesson 15 into MVVM:

1. Create `@Observable final class TipViewModel` with `amount`, `percent` and `roundUp`, and computed `tip` and `total`.
2. The view owns it with `@State` and binds controls with `$viewModel.amount` and so on.
3. Extract the picker into a `TipPicker` that takes `@Bindable var viewModel: TipViewModel`.
:::

:::solution Show solution
```swift
import SwiftUI

@Observable
final class TipViewModel {
    var amount = 0.0
    var percent = 15
    var roundUp = false

    var tip: Double {
        let raw = amount * Double(percent) / 100
        return roundUp ? raw.rounded(.up) : raw
    }
    var total: Double { amount + tip }
}

struct TipPicker: View {
    @Bindable var viewModel: TipViewModel
    var body: some View {
        Picker("Tip", selection: $viewModel.percent) {
            ForEach([10, 15, 20], id: \.self) { Text("\($0)%").tag($0) }
        }
        .pickerStyle(.segmented)
    }
}

struct TipScreen: View {
    @State private var viewModel = TipViewModel()

    var body: some View {
        Form {
            TextField("Amount", value: $viewModel.amount, format: .number)
                .keyboardType(.decimalPad)
            TipPicker(viewModel: viewModel)
            Toggle("Round up", isOn: $viewModel.roundUp)
            LabeledContent("Tip", value: viewModel.tip.formatted(.currency(code: "USD")))
            LabeledContent("Total", value: viewModel.total.formatted(.currency(code: "USD")))
        }
    }
}

#Preview { TipScreen() }
```
:::

## Check your understanding

```quiz
Q: How should a view create and own an @Observable view model?
- [ ] `let viewModel = MyViewModel()`
- [x] `@State private var viewModel = MyViewModel()`
- [ ] `@Binding var viewModel: MyViewModel`
- [ ] `@Published var viewModel`
> @State keeps the instance alive across view updates.

Q: A child view needs a binding to a property of an @Observable object it receives. What does it use?
- [ ] @State
- [ ] @Environment
- [x] @Bindable
- [ ] @Binding
> @Bindable enables $model.property bindings on an observable object.

Q: Why model loading as `enum LoadState { case loading, loaded(T), failed(String) }` instead of several Booleans?
- [ ] Enums are faster
- [x] Impossible combinations, like loading and failed at the same time, can't be represented
- [ ] Booleans can't be observed
- [ ] SwiftUI requires enums
> One enum value means exactly one state at a time.

Q: Why mark a view model `@MainActor`?
- [ ] To make network calls run on the main thread
- [x] Because its state drives the UI, which must be updated on the main actor
- [ ] To make it a singleton
- [ ] It's required for @Observable
> UI state belongs on the main actor; awaited work still doesn't block it.
```
