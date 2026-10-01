---
id: setup
title: Install Xcode & Run Your First App
part: 1
minutes: 30
summary: Install Xcode, create a SwiftUI project, run it in the iPhone Simulator and on your own iPhone, and see changes instantly with live previews.
---

:::goals
- How to install Xcode and the iOS platform
- How to create a new SwiftUI app project
- How to run your app in the Simulator *and* on a real iPhone
- How to use SwiftUI previews to see changes instantly
- The Xcode shortcuts you'll use every day
:::

## 1. Install Xcode

1. Open the **App Store** on your Mac, search for **Xcode** and click **Get**. It's a large download (well over 10 GB once installed), so use a good connection and make sure you have plenty of free disk space.
   - Alternatively, download it from [developer.apple.com/xcode](https://developer.apple.com/xcode/). You can also find beta versions there.
2. Launch Xcode. On first launch it asks which **platforms** to install. Make sure **iOS** is selected (it includes the iPhone Simulator).
3. Let it finish installing components. Grab a coffee. ☕

:::note System requirements
Xcode 27 needs **macOS Tahoe 26.6 or later**. 16 GB of RAM is comfortable, and **keep at least 40–50 GB free**: Xcode, simulators and build caches grow quickly. You can reclaim space later from **Xcode → Settings → Components** and by deleting old simulators.
:::

## 2. Create your first project

1. In the Welcome window click **Create New Project…** (or **File → New → Project**).
2. Choose the **iOS** tab → **App** → **Next**.
3. Fill in the options:

| Field | Value | What it means |
|---|---|---|
| Product Name | `HelloiOS` | The app's name |
| Team | None (for now) | Your signing team. We'll set it when running on a device |
| Organization Identifier | `com.yourname` | Usually your reversed domain |
| Bundle Identifier | `com.yourname.HelloiOS` | Generated automatically. Your app's unique ID, which you can't change after publishing |
| Interface | **SwiftUI** | The modern UI framework |
| Language | **Swift** | |
| Testing System | Swift Testing | The modern testing framework (Lesson 31) |
| Storage | None | We'll add SwiftData later |

4. Click **Next**, choose where to save, and click **Create**.

## 3. Run it in the Simulator ▶

At the top of the Xcode window there's a **run destination** menu. Choose a recent **iPhone** simulator, then click the **▶ Run** button, or press [[Cmd]] + [[R]].

Xcode builds the app, boots the Simulator (a virtual iPhone on your Mac) and launches it. You'll see a globe icon and **"Hello, world!"**. Congratulations, you've just run your first iOS app! 🎉

:::tip Simulator tips
[[Cmd]] + [[Shift]] + [[H]] goes to the Home Screen, [[Cmd]] + [[←]] / [[→]] rotates, [[Cmd]] + [[Shift]] + [[A]] toggles dark mode, and [[Cmd]] + [[S]] takes a screenshot. Use **Features** and **I/O** in the Simulator's menu bar for things like Face ID, location and the keyboard.
:::

## 4. Look at the code

Xcode created two Swift files. First, the app's entry point:

```swift title="HelloiOSApp.swift"
import SwiftUI

@main                                  // 1. where the app starts
struct HelloiOSApp: App {              // 2. the app itself
    var body: some Scene {
        WindowGroup {                  // 3. the app's window(s)
            ContentView()              // 4. the first screen
        }
    }
}
```

1. `@main` marks the type where the program begins.
2. The app is a struct conforming to the `App` protocol.
3. A `WindowGroup` scene manages your app's window. On iPad and Mac there can be several.
4. `ContentView()` is the first view shown.

And the first screen:

```swift title="ContentView.swift"
import SwiftUI

struct ContentView: View {             // 1. a view is a struct conforming to View
    var body: some View {              // 2. body describes what it looks like
        VStack {                       // 3. a vertical stack
            Image(systemName: "globe") // 4. an SF Symbols icon
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text("Hello, world!")
        }
        .padding()                     // 5. a modifier
    }
}

#Preview {                             // 6. a live preview in Xcode's canvas
    ContentView()
}
```

1. SwiftUI screens are small **structs** that conform to `View`.
2. `body` returns what the view looks like. SwiftUI calls it whenever it needs to redraw.
3. `VStack` stacks its children vertically.
4. `Image(systemName:)` shows one of Apple's thousands of built-in **SF Symbols** icons.
5. **Modifiers** like `.padding()` customise a view. You'll use them everywhere.
6. `#Preview` renders the view in Xcode's **canvas**, with no Simulator needed.

## 5. Live previews

Open `ContentView.swift`. If the canvas isn't visible on the right, press [[Option]] + [[Cmd]] + [[Enter]]. If the preview is paused, press [[Option]] + [[Cmd]] + [[P]] to resume.

Now change `"Hello, world!"` to `"Hello, <your name>!"` and watch the preview update as you type. Previews are interactive: buttons work, and you can switch to dark mode or different devices using the controls under the preview.

## 6. Run on your own iPhone (optional but recommended)

1. In Xcode, open **Xcode → Settings → Accounts**, click **+** and sign in with your Apple Account.
2. Select the project in the navigator (the blue icon at the top), then the **HelloiOS** target → **Signing & Capabilities**. Tick **Automatically manage signing** and choose your **Team** (your name, *Personal Team*).
3. Connect your iPhone with a cable (after the first time, it also works over Wi-Fi) and tap **Trust** on the phone.
4. On the iPhone, enable **Settings → Privacy & Security → Developer Mode** and restart when asked.
5. Choose your iPhone as the run destination and press [[Cmd]] + [[R]].
6. With a free account, the first launch may be blocked: on the iPhone open **Settings → General → VPN & Device Management**, tap your developer profile and choose **Trust**.

:::note Free accounts
Apps installed with a free Personal Team expire after **7 days**. Just run them from Xcode again. The paid Developer Program removes this limit and unlocks TestFlight and the App Store.
:::

## Shortcuts worth learning now

| Action | Shortcut |
|---|---|
| Run | [[Cmd]] + [[R]] |
| Stop | [[Cmd]] + [[.]] |
| Build | [[Cmd]] + [[B]] |
| Clean build folder | [[Cmd]] + [[Shift]] + [[K]] |
| Open any file or symbol quickly | [[Cmd]] + [[Shift]] + [[O]] |
| Show/hide canvas | [[Option]] + [[Cmd]] + [[Enter]] |
| Resume preview | [[Option]] + [[Cmd]] + [[P]] |
| Library of views, modifiers, symbols | [[Cmd]] + [[Shift]] + [[L]] |
| Re-indent selection | [[Ctrl]] + [[I]] |
| Comment/uncomment | [[Cmd]] + [[/]] |
| Quick Help for a symbol | [[Option]] + click |
| Jump to definition | [[Cmd]] + click |
| Show fix-it / code actions | [[Cmd]] + [[Shift]] + [[A]] |

:::tip Read the red and yellow
Xcode shows errors (red) and warnings (yellow) inline. Click the icon to see the message, and often a **Fix** button that corrects the code for you.
:::

## Troubleshooting

- **"No such module 'SwiftUI'" or strange build errors after an update**: **Product → Clean Build Folder** ([[Cmd]] + [[Shift]] + [[K]]) and build again.
- **No simulators in the destination menu**: open **Xcode → Settings → Components** and install the iOS platform.
- **Preview failed**: click **Diagnostics** on the error banner; it's usually a compile error elsewhere in the file. Fix it, then resume ([[Option]] + [[Cmd]] + [[P]]).
- **"Untrusted Developer" on the iPhone**: trust your profile in Settings → General → VPN & Device Management.
- **Signing errors**: make sure your Team is selected and the bundle identifier is unique (add your name to it).

:::exercise
1. Change the text to `"Hello, <your name>! Welcome to iOS."` and the symbol to `"swift"` (the Swift bird).
2. Add a second `#Preview("Dark") { ContentView().preferredColorScheme(.dark) }` and look at both previews.
3. Run the app in the Simulator and, if you can, on your iPhone.
:::

:::solution Show solution
```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Image(systemName: "swift")
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text("Hello, Ada! Welcome to iOS.")
        }
        .padding()
    }
}

#Preview("Light") {
    ContentView()
}

#Preview("Dark") {
    ContentView()
        .preferredColorScheme(.dark)
}
```
Each `#Preview` appears as a tab in the canvas. Give them names to tell them apart.
:::

## Check your understanding

```quiz
Q: What does the `@main` attribute do?
- [ ] Marks the main screen of the app
- [x] Marks the type where the program starts
- [ ] Makes a view appear in previews
- [ ] Runs the app on a device
> Exactly one type in the app is marked @main. For SwiftUI apps, it's your App struct.

Q: Which property describes what a SwiftUI view looks like?
- [ ] `view`
- [ ] `render()`
- [x] `body`
- [ ] `layout`
> Every View has a `body` that returns other views.

Q: What is `#Preview` for?
- [ ] It runs the app in the Simulator
- [x] It renders a view live in Xcode's canvas while you edit
- [ ] It's required for the app to compile
- [ ] It uploads a screenshot to the App Store
> Previews are a development tool. They don't affect the running app.

Q: What do you need to run an app on your own iPhone?
- [ ] A paid Developer Program membership
- [x] An Apple Account added to Xcode, a selected Team, and Developer Mode enabled on the phone
- [ ] A jailbroken iPhone
- [ ] TestFlight
> A free Personal Team is enough for your own devices (apps expire after 7 days).
```
