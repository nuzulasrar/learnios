---
id: storage
title: Settings, Files & the Keychain
part: 5
minutes: 25
summary: Not everything belongs in a database. Store preferences with @AppStorage and UserDefaults, read and write files, and keep secrets safe in the Keychain.
---

:::goals
- Choosing the right storage for each kind of data
- `UserDefaults` and `@AppStorage` for small settings
- A working in-app appearance setting (System / Light / Dark)
- Reading and writing files in Documents and Caches
- Storing tokens and passwords securely in the Keychain
:::

## Choosing storage

| Data | Where | Why |
|---|---|---|
| Small settings and flags ("dark mode", "has seen onboarding") | `UserDefaults` / `@AppStorage` | Simple key–value storage |
| Small UI state to restore (selected tab, draft) | `@SceneStorage` | Per-window state restoration (Lesson 19) |
| Your app's structured data | SwiftData | Queries, relationships, live updates |
| Downloaded JSON, images, exports | Files (Caches / Documents) | Large or file-shaped data |
| Passwords, tokens, API keys | **Keychain** | Encrypted, and can survive reinstalls |

## `@AppStorage`

`@AppStorage` reads and writes `UserDefaults` and updates your views automatically, like `@State` that persists across launches:

```swift title="SettingsView.swift"
import SwiftUI

enum Appearance: String, CaseIterable, Identifiable {
    case system, light, dark
    var id: Self { self }

    var colorScheme: ColorScheme? {
        switch self {
        case .system: nil            // follow the system setting
        case .light: .light
        case .dark: .dark
        }
    }
}

struct SettingsView: View {
    @AppStorage("appearance") private var appearance = Appearance.system
    @AppStorage("dailyTips") private var dailyTips = true
    @AppStorage("servingsDefault") private var servings = 2

    var body: some View {
        Form {
            Picker("Appearance", selection: $appearance) {
                ForEach(Appearance.allCases) { Text($0.rawValue.capitalized).tag($0) }
            }
            Toggle("Daily tips", isOn: $dailyTips)
            Stepper("Default servings: \(servings)", value: $servings, in: 1...12)
        }
    }
}

#Preview { SettingsView() }
```

`@AppStorage` supports `Bool`, `Int`, `Double`, `String`, `URL`, `Data`, and enums backed by `String` or `Int` raw values, like `Appearance` above.

### Applying the appearance app-wide

Read the same key at the top of the app and apply it with `.preferredColorScheme`:

```swift title="SettingsApp.swift"
import SwiftUI

struct RootView: View {
    @AppStorage("appearance") private var appearance = Appearance.system

    var body: some View {
        NavigationStack {
            SettingsView().navigationTitle("Settings")
        }
        .preferredColorScheme(appearance.colorScheme)   // nil = follow the system
    }
}

#Preview { RootView() }
```

Change the picker, and the whole app switches instantly. It's remembered after restart too. 🌙

:::warning Keep UserDefaults small
`UserDefaults` is loaded into memory at launch. Store small values only, never large arrays or images, and **never** passwords or tokens, since UserDefaults isn't encrypted.
:::

## Files

Each app has a sandboxed container with standard folders:

- **Documents**: user-created content that should be backed up.
- **Caches**: re-downloadable data (API responses, images). The system may delete it when storage is low.
- **Application Support**: app-managed files that should persist but aren't user documents.

```swift runnable
import Foundation

struct Settings: Codable {
    var favoriteCuisine: String
    var lastOpened: Date
}

let folder = URL.cachesDirectory                 // also: .documentsDirectory, .applicationSupportDirectory
let file = folder.appending(path: "settings.json")

do {
    // Write
    let data = try JSONEncoder().encode(Settings(favoriteCuisine: "Italian", lastOpened: .now))
    try data.write(to: file, options: .atomic)   // atomic: never leaves a half-written file

    // Read
    let loaded = try JSONDecoder().decode(Settings.self, from: Data(contentsOf: file))
    print("Loaded:", loaded.favoriteCuisine)

    // Check, list, delete
    print(FileManager.default.fileExists(atPath: file.path()))
    try FileManager.default.removeItem(at: file)
    print(FileManager.default.fileExists(atPath: file.path()))
} catch {
    print("File error:", error)
}
```

Codable + files is a simple, powerful combination. You'll use it for offline caching in the next lesson.

## The Keychain

The **Keychain** is the system's encrypted store for secrets: passwords, login tokens, API keys. Its API is a low-level C-style interface, so apps wrap it in a small helper:

```swift title="KeychainStore.swift"
import Foundation
import Security

enum KeychainStore {
    static func save(_ value: String, for key: String) throws {
        let data = Data(value.utf8)
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        SecItemDelete(query as CFDictionary)                  // replace any existing item
        var attributes = query
        attributes[kSecValueData as String] = data
        attributes[kSecAttrAccessible as String] = kSecAttrAccessibleAfterFirstUnlock
        let status = SecItemAdd(attributes as CFDictionary, nil)
        guard status == errSecSuccess else { throw KeychainError(status: status) }
    }

    static func read(_ key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        var result: AnyObject?
        guard SecItemCopyMatching(query as CFDictionary, &result) == errSecSuccess,
              let data = result as? Data else { return nil }
        return String(decoding: data, as: UTF8.self)
    }

    static func delete(_ key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]
        SecItemDelete(query as CFDictionary)
    }
}

struct KeychainError: Error { let status: OSStatus }
```

```swift
try KeychainStore.save(token, for: "authToken")      // after logging in
let token = KeychainStore.read("authToken")          // when making requests
KeychainStore.delete("authToken")                    // on sign-out
```

:::danger Never put secrets in your app's code
Anything compiled into your app, including API keys in Swift files or Info.plist, can be extracted by anyone who downloads it. Real secrets belong on a server. The Keychain is for secrets that belong to **the user**, like their login token.
:::

## Recap

:::recap
- `@AppStorage` for small settings, which updates views automatically; `.preferredColorScheme` applies the appearance.
- Files: `URL.cachesDirectory` / `.documentsDirectory` + Codable + atomic writes.
- Keychain for tokens and passwords; never UserDefaults for secrets.
:::

:::exercise
1. Add an **onboarding** screen shown only on first launch: use `@AppStorage("hasSeenOnboarding")` and show it with `.fullScreenCover(isPresented:)`. A "Get started" button sets the flag.
2. Save and load an array of `Codable` recent searches to a JSON file in Application Support, keeping only the 10 most recent.
:::

## Check your understanding

```quiz
Q: Where should you store a user's login token?
- [ ] UserDefaults
- [ ] A text file in Documents
- [x] The Keychain
- [ ] @AppStorage
> The Keychain is encrypted and designed for secrets.

Q: What does `@AppStorage("appearance")` do?
- [ ] Stores the value in memory only
- [x] Reads and writes UserDefaults under that key, and updates views when it changes
- [ ] Syncs to a server
- [ ] Stores it in SwiftData
> It's a SwiftUI-friendly wrapper around UserDefaults.

Q: Which folder is right for re-downloadable API responses?
- [ ] Documents
- [x] Caches
- [ ] The app bundle
- [ ] Keychain
> The system may clear Caches, which is fine for data you can fetch again.

Q: Why write files with `.atomic`?
- [ ] It compresses the file
- [x] The write goes to a temporary file first, so a crash never leaves a half-written file
- [ ] It encrypts the file
- [ ] It's faster
> Atomic writes protect against corrupted data.
```
