---
id: swiftui-lists
title: Lists, Grids & Forms
part: 3
minutes: 35
summary: Almost every app shows lists of data. Learn List and ForEach, identity, sections, swipe actions, editing, lazy stacks and grids, and building settings screens with Form.
---

:::goals
- `List` and `ForEach` with `Identifiable` data
- Why stable **identity** matters
- Sections, swipe actions, delete and move
- `LazyVStack` and `LazyVGrid` for custom scrolling layouts
- `Form` for settings and input screens
:::

## `List`

`List` shows scrollable rows in the standard iOS style, and only creates the rows that are on screen, so it handles thousands of items.

```swift title="FruitList.swift"
import SwiftUI

struct FruitList: View {
    let fruits = ["Apple", "Banana", "Cherry", "Mango"]

    var body: some View {
        List(fruits, id: \.self) { fruit in      // id: tells SwiftUI how to identify each row
            Text(fruit)
        }
    }
}

#Preview { FruitList() }
```

## Identity and `Identifiable`

SwiftUI must know which row is which, so it can animate insertions, keep row state attached to the right item, and update efficiently. The best way is to make your model **`Identifiable`**, with a stable, unique `id`:

```swift
struct Task: Identifiable {
    let id = UUID()          // unique and stable for the lifetime of the item
    var title: String
    var isDone = false
}

List(tasks) { task in        // no id: needed, since Identifiable provides it
    Text(task.title)
}
```

:::danger Don't use array indices as IDs
`ForEach(items.indices, id: \.self)` breaks when items are inserted or removed: rows get mixed up and animations glitch. Use a real `id`, like a database or server ID, or a `UUID` created once with the item.
:::

## A complete task list

This example has sections, a static row, swipe actions, delete and reordering. Type it in and run it:

```swift title="TaskListView.swift"
import SwiftUI

struct TodoItem: Identifiable {
    let id = UUID()
    var title: String
    var isDone = false
}

struct TaskListView: View {
    @State private var items = [
        TodoItem(title: "Buy flour"),
        TodoItem(title: "Preheat oven"),
        TodoItem(title: "Invite friends", isDone: true)
    ]
    @State private var newTitle = ""

    var body: some View {
        NavigationStack {
            List {
                Section {
                    HStack {
                        TextField("New task", text: $newTitle)
                            .onSubmit(add)
                        Button("Add", action: add)
                            .disabled(newTitle.isEmpty)
                    }
                }

                Section("To do (\(items.filter { !$0.isDone }.count))") {
                    ForEach($items) { $item in                // bindings to each element!
                        Toggle(isOn: $item.isDone) {
                            Text(item.title)
                                .strikethrough(item.isDone)
                                .foregroundStyle(item.isDone ? .secondary : .primary)
                        }
                        .swipeActions(edge: .leading) {
                            Button("Duplicate") {
                                items.append(TodoItem(title: item.title))
                            }
                            .tint(.blue)
                        }
                    }
                    .onDelete { offsets in items.remove(atOffsets: offsets) }   // swipe to delete
                    .onMove { from, to in items.move(fromOffsets: from, toOffset: to) }
                }
            }
            .navigationTitle("Tasks")
            .toolbar { EditButton() }                 // enables reordering
            .animation(.default, value: items.count)
        }
    }

    private func add() {
        let title = newTitle.trimmingCharacters(in: .whitespaces)
        guard !title.isEmpty else { return }
        items.append(TodoItem(title: title))
        newTitle = ""
    }
}

#Preview { TaskListView() }
```

Things to notice:

- `List { … }` can mix static rows and `ForEach` for dynamic ones.
- **`ForEach($items) { $item in … }`** gives you a *binding* to each element, so the `Toggle` can change `item.isDone` directly in the array.
- `.onDelete` adds swipe-to-delete; `.onMove` + `EditButton` enable drag-to-reorder.
- `.swipeActions` adds custom swipe buttons.

## List styles and extras

```swift
List { … }
    .listStyle(.insetGrouped)       // .plain, .grouped, .insetGrouped, .sidebar
    .refreshable {                  // pull to refresh (async)
        await reload()
    }
    .overlay {
        if items.isEmpty {
            ContentUnavailableView("No tasks", systemImage: "checklist",
                                   description: Text("Add your first task above."))
        }
    }
```

`ContentUnavailableView` is Apple's standard **empty state**. There's even a built-in search version: `ContentUnavailableView.search(text: query)`.

## Lazy stacks and grids

When you want a custom look instead of standard list rows, use a `ScrollView` with **lazy** stacks or grids. They also create only the views that are visible:

```swift title="PhotoGrid.swift"
import SwiftUI

struct PhotoGrid: View {
    let columns = [GridItem(.adaptive(minimum: 110), spacing: 12)]   // as many columns as fit

    var body: some View {
        ScrollView {
            LazyVGrid(columns: columns, spacing: 12) {
                ForEach(1...30, id: \.self) { i in
                    RoundedRectangle(cornerRadius: 12)
                        .fill(Color(hue: Double(i) / 30, saturation: 0.6, brightness: 0.9))
                        .aspectRatio(1, contentMode: .fit)
                        .overlay(Text("\(i)").font(.title2.bold()).foregroundStyle(.white))
                }
            }
            .padding()
        }
    }
}

#Preview { PhotoGrid() }
```

| Column type | Behaviour |
|---|---|
| `GridItem(.flexible())` × N | N equal columns |
| `GridItem(.fixed(100))` | Exactly 100 points wide |
| `GridItem(.adaptive(minimum: 110))` | As many columns as fit, which suits phones and iPads alike |

For horizontal carousels use `ScrollView(.horizontal) { LazyHStack { … } }`, optionally with `.scrollTargetBehavior(.viewAligned)` for snapping.

## Forms

`Form` is a `List` styled for settings and data entry. Controls inside it automatically get the right iOS appearance:

```swift title="ProfileSettings.swift"
import SwiftUI

struct ProfileSettings: View {
    @State private var name = "Ada"
    @State private var notifications = true
    @State private var theme = "System"
    @State private var birthday = Date.now

    var body: some View {
        NavigationStack {
            Form {
                Section("Profile") {
                    TextField("Name", text: $name)
                    DatePicker("Birthday", selection: $birthday, displayedComponents: .date)
                }
                Section {
                    Toggle("Notifications", isOn: $notifications)
                    Picker("Theme", selection: $theme) {
                        ForEach(["System", "Light", "Dark"], id: \.self) { Text($0) }
                    }
                } header: {
                    Text("Preferences")
                } footer: {
                    Text("Notifications suggest a recipe every morning.")
                }
                Section {
                    Button("Sign out", role: .destructive) { }
                }
            }
            .navigationTitle("Settings")
        }
    }
}

#Preview { ProfileSettings() }
```

## Recap

:::recap
- `List` and `ForEach` need stable identity: make models `Identifiable`.
- `ForEach($items) { $item in }` gives bindings to elements.
- `.onDelete`, `.onMove`, `.swipeActions`, `.refreshable`, `ContentUnavailableView`.
- `LazyVGrid` with adaptive columns for custom grids; `Form` for settings.
:::

## Practice

:::exercise
1. Add a **filter** to `TaskListView`: a segmented picker (All / To do / Done) above the list, showing only matching tasks. (Hint: filter a *computed* array, and use `ForEach` over the filtered items with bindings looked up by id, or toggle via a function.)
2. Show `ContentUnavailableView` when the filtered list is empty.
:::

:::solution Show solution
```swift
import SwiftUI

struct TodoItem: Identifiable {
    let id = UUID()
    var title: String
    var isDone = false
}

enum TaskFilter: String, CaseIterable, Identifiable {
    case all = "All", todo = "To do", done = "Done"
    var id: Self { self }
}

struct FilteredTasksView: View {
    @State private var items = [TodoItem(title: "Buy flour"), TodoItem(title: "Bake", isDone: true)]
    @State private var filter = TaskFilter.all

    private var visible: [TodoItem] {
        switch filter {
        case .all: items
        case .todo: items.filter { !$0.isDone }
        case .done: items.filter(\.isDone)
        }
    }

    var body: some View {
        NavigationStack {
            List(visible) { item in
                Button {
                    toggle(item)
                } label: {
                    Label(item.title, systemImage: item.isDone ? "checkmark.circle.fill" : "circle")
                }
            }
            .overlay {
                if visible.isEmpty {
                    ContentUnavailableView("Nothing here", systemImage: "tray")
                }
            }
            .safeAreaInset(edge: .top) {
                Picker("Filter", selection: $filter) {
                    ForEach(TaskFilter.allCases) { Text($0.rawValue).tag($0) }
                }
                .pickerStyle(.segmented)
                .padding(.horizontal)
            }
            .navigationTitle("Tasks")
        }
    }

    private func toggle(_ item: TodoItem) {
        guard let index = items.firstIndex(where: { $0.id == item.id }) else { return }
        items[index].isDone.toggle()
    }
}

#Preview { FilteredTasksView() }
```
Note the pattern: the **source of truth** is `items`; `visible` is derived; changes go through a function that updates the source by `id`.
:::

## Check your understanding

```quiz
Q: Why should list data be Identifiable with stable IDs?
- [ ] It makes lists sort automatically
- [x] So SwiftUI can track which row is which across updates, keeping state and animations correct
- [ ] It's required for text to display
- [ ] It saves memory
> Stable identity is how SwiftUI matches old rows to new rows.

Q: What does `ForEach($items) { $item in ... }` give you?
- [ ] A copy of each item
- [x] A binding to each element, so controls can edit items in place
- [ ] The index of each item
- [ ] A sorted array
> Binding-based ForEach lets a Toggle or TextField modify the array directly.

Q: Which grid column type fits as many columns as the width allows?
- [ ] .fixed
- [ ] .flexible
- [x] .adaptive(minimum:)
- [ ] .automatic
> Adaptive columns work well across iPhone and iPad sizes.

Q: What's the standard way to show an empty state?
- [ ] An alert
- [x] ContentUnavailableView
- [ ] An empty List
- [ ] ProgressView
> ContentUnavailableView gives the familiar iOS empty-state look.
```
