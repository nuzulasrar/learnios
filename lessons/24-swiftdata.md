---
id: swiftdata
title: Persisting Data with SwiftData
part: 5
minutes: 40
summary: Save your app's data on the device with SwiftData. Define models with @Model, query them live with @Query, insert, update and delete, and use relationships and in-memory previews.
---

:::goals
- When you need a database
- Defining models with `@Model`
- Setting up a `modelContainer`
- Reading with `@Query` (sorting and filtering with `#Predicate`)
- Inserting, updating and deleting with `modelContext`
- Relationships, previews with in-memory data, and schema changes
:::

## Why SwiftData?

Use a database when you need to store **structured data** that must survive app restarts: notes, tasks, favourites, workouts. **SwiftData** is Apple's modern persistence framework. It's built on the battle-tested Core Data engine (SQLite underneath) but designed for Swift and SwiftUI:

- Models are plain Swift classes marked `@Model`.
- `@Query` in a view **automatically updates** the UI when data changes.
- It can sync through iCloud with a capability switch.

## 1. Define a model

```swift title="Note.swift" nocheck
import Foundation
import SwiftData

@Model
final class Note {
    var title: String
    var body: String
    var isPinned: Bool
    var createdAt: Date

    init(title: String, body: String = "", isPinned: Bool = false, createdAt: Date = .now) {
        self.title = title
        self.body = body
        self.isPinned = isPinned
        self.createdAt = createdAt
    }
}
```

`@Model` turns the class into a persistent, observable model. Every stored property becomes a column. Models must be classes, and need an initialiser.

## 2. Set up the container

Tell your app which models to store, once, at the top:

```swift title="NotesApp.swift" nocheck
import SwiftUI
import SwiftData

@main
struct NotesApp: App {
    var body: some Scene {
        WindowGroup {
            NotesListView()
        }
        .modelContainer(for: Note.self)      // creates/opens the database
    }
}
```

That's all the setup. The container creates the database file in your app's sandbox and puts a **model context** into the SwiftUI environment.

## 3. Query, insert, update and delete

```swift title="NotesListView.swift" nocheck
import SwiftUI
import SwiftData

struct NotesListView: View {
    @Environment(\.modelContext) private var context              // for writes
    @Query(sort: [SortDescriptor(\Note.isPinned, order: .reverse),
                  SortDescriptor(\Note.createdAt, order: .reverse)])
    private var notes: [Note]                                       // live results
    @State private var newTitle = ""

    var body: some View {
        NavigationStack {
            List {
                Section {
                    HStack {
                        TextField("New note", text: $newTitle)
                            .onSubmit(addNote)
                        Button("Add", action: addNote).disabled(newTitle.isEmpty)
                    }
                }
                ForEach(notes) { note in
                    NavigationLink(value: note) {
                        Label(note.title, systemImage: note.isPinned ? "pin.fill" : "note.text")
                    }
                    .swipeActions(edge: .leading) {
                        Button(note.isPinned ? "Unpin" : "Pin") {
                            note.isPinned.toggle()                  // just change it: saved automatically
                        }
                        .tint(.orange)
                    }
                }
                .onDelete { offsets in
                    for index in offsets { context.delete(notes[index]) }
                }
            }
            .navigationTitle("Notes")
            .navigationDestination(for: Note.self) { note in
                NoteEditor(note: note)
            }
            .overlay {
                if notes.isEmpty {
                    ContentUnavailableView("No notes", systemImage: "note.text",
                                           description: Text("Add your first note above."))
                }
            }
        }
    }

    private func addNote() {
        context.insert(Note(title: newTitle))                        // insert
        newTitle = ""
    }
}

struct NoteEditor: View {
    @Bindable var note: Note                                         // models are observable

    var body: some View {
        Form {
            TextField("Title", text: $note.title)
            TextField("Body", text: $note.body, axis: .vertical)
                .lineLimit(5...)
            Toggle("Pinned", isOn: $note.isPinned)
        }
        .navigationTitle(note.title)
    }
}

#Preview {
    NotesListView()
        .modelContainer(for: Note.self, inMemory: true)              // throwaway data for previews
}
```

Look how little code there is:

- **`@Query`** fetches notes, sorted, and **re-runs automatically** whenever notes are inserted, changed or deleted. No manual refreshing.
- **Insert** with `context.insert(_:)`, **delete** with `context.delete(_:)`.
- **Update** by simply changing a property. Models are observable, and SwiftData **autosaves**.
- `@Bindable var note` lets the editor bind text fields directly to the model.

<div class="diagram"><div class="flow">
<div>User action<small>insert / edit / delete</small></div><div class="arrow">→</div>
<div class="hl">modelContext</div><div class="arrow">→</div>
<div>Database</div><div class="arrow">→</div>
<div class="hl">@Query re-runs → UI updates</div>
</div></div>

## Filtering with `#Predicate`

```swift nocheck
@Query(filter: #Predicate<Note> { $0.isPinned },
       sort: \Note.createdAt, order: .reverse)
private var pinnedNotes: [Note]
```

For a filter that depends on user input (like a search box), create the query in the initialiser of a subview:

```swift title="SearchResults.swift" nocheck
struct SearchResults: View {
    @Query private var notes: [Note]

    init(searchText: String) {
        _notes = Query(filter: #Predicate<Note> { note in
            searchText.isEmpty || note.title.localizedStandardContains(searchText)
        }, sort: \Note.createdAt)
    }

    var body: some View {
        List(notes) { Text($0.title) }
    }
}
```

## Relationships

Models can reference each other. SwiftData infers relationships from the property types:

```swift nocheck
@Model
final class Notebook {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \Page.notebook)
    var pages: [Page] = []                    // one notebook → many pages
    init(name: String) { self.name = name }
}

@Model
final class Page {
    var text: String
    var notebook: Notebook?                   // each page belongs to a notebook
    init(text: String) { self.text = text }
}

// Usage:
let notebook = Notebook(name: "Recipes")
context.insert(notebook)
notebook.pages.append(Page(text: "Grandma's lasagne"))
```

`deleteRule: .cascade` deletes the pages when their notebook is deleted.

## Unique values and useful attributes

```swift nocheck
@Model
final class FavoriteRecipe {
    @Attribute(.unique) var recipeID: Int     // no duplicates: inserting the same id updates instead
    var name: String
    @Attribute(.externalStorage) var photo: Data?   // large data stored outside the database
    @Transient var isSelected = false               // not saved
    init(recipeID: Int, name: String) { self.recipeID = recipeID; self.name = name }
}
```

## Changing your models later

When you add a new property with a **default value**, or add a new model, SwiftData migrates existing databases automatically (a *lightweight migration*). Renaming or changing types needs a `VersionedSchema` and a `SchemaMigrationPlan`, which is an advanced topic you'll look up when you need it. During early development you can also delete the app from the Simulator to start fresh.

:::warning Existing users keep their data
Once your app is on the App Store, users have databases on their phones. Test that an update opens old data correctly before you release it.
:::

## Recap

:::recap
- `@Model final class` defines what's stored; `.modelContainer(for:)` sets up the database.
- `@Query` reads (sorted, filtered with `#Predicate`) and updates the UI automatically.
- `modelContext.insert` / `.delete`; change properties directly to update. SwiftData autosaves.
- `@Bindable` binds forms to models; previews use `inMemory: true`.
:::

:::exercise
Extend the notes app:

1. Add a `category: String` property with a default value (`"General"`) and a `Picker` in the editor.
2. Add `.searchable` to the list that filters by title using a `SearchResults`-style subview.
3. Add a toolbar menu to sort by date or by title. (Hint: keep the sort choice in `@State` and pass `SortDescriptor`s to the subview's `Query`.)
:::

## Check your understanding

```quiz
Q: How do you save changes after editing a property of a SwiftData model?
- [ ] Call context.save() after every change, or nothing is stored
- [x] Just change the property. SwiftData autosaves (you can still call save() explicitly)
- [ ] Delete and re-insert the object
- [ ] Write it to UserDefaults
> Models are tracked by their context and saved automatically.

Q: What does `@Query` do?
- [ ] Fetches data once when the view appears
- [x] Fetches models from the store and keeps the view updated as the data changes
- [ ] Sends a network request
- [ ] Creates the database
> @Query is a live, observed fetch.

Q: How should previews that use SwiftData be set up?
- [ ] They can't use SwiftData
- [x] With `.modelContainer(for: Model.self, inMemory: true)` so they use throwaway data
- [ ] By copying the real database file
- [ ] With UserDefaults
> In-memory containers keep previews fast and isolated.

Q: What does `@Attribute(.unique)` on `recipeID` do?
- [ ] Encrypts the value
- [x] Ensures no two models share that value; inserting a duplicate updates the existing one
- [ ] Makes it optional
- [ ] Excludes it from saving
> Unique attributes give upsert behaviour.
```
