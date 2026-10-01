---
id: xcode-tour
title: Tour of Xcode & Your Project
part: 1
minutes: 25
summary: Find your way around Xcode, understand every file in a new project, and learn about targets, build settings, Info.plist, asset catalogs and Swift packages.
---

:::goals
- The main areas of the Xcode window
- What each file in a new project is for
- Targets, schemes and build settings
- Version and build numbers, deployment target, Info.plist and capabilities
- Asset catalogs, String Catalogs and adding Swift packages
:::

## The Xcode window

<div class="diagram">
<div class="flow">
<div>Navigator<small>left: files, search, issues, tests, debug</small></div>
<div class="hl">Editor<small>centre: code + canvas previews</small></div>
<div>Inspectors<small>right: file info, Quick Help, attributes</small></div>
</div>
<div class="flow" style="margin-top:8px">
<div>Toolbar<small>top: run/stop, scheme, destination, status</small></div>
<div>Debug area<small>bottom: console output and variables</small></div>
</div>
</div>

Toggle panels with [[Cmd]] + [[0]] (navigator), [[Cmd]] + [[Option]] + [[0]] (inspectors) and [[Cmd]] + [[Shift]] + [[Y]] (debug area). The navigator's tabs are [[Cmd]] + [[1]] to [[Cmd]] + [[9]]: [[Cmd]] + [[1]] is files, [[Cmd]] + [[3]] find, [[Cmd]] + [[5]] issues, [[Cmd]] + [[6]] tests.

## Your project's files

<div class="tree">HelloiOS/
├── HelloiOS.xcodeproj           ← the project: targets, settings, file references
├── HelloiOS/                    ← your app's source folder
│   ├── HelloiOSApp.swift        ← @main App entry point
│   ├── ContentView.swift        ← the first screen
│   └── Assets.xcassets          ← asset catalog: app icon, accent colour, images, colours
├── HelloiOSTests/               ← unit tests (Swift Testing)
└── HelloiOSUITests/             ← UI tests (XCTest)</div>

Recent Xcode versions keep the navigator **in sync with the folder on disk**: create a folder in Finder or Xcode and it shows up in both. Organise code into folders like `Models`, `Views` and `Services` as the app grows.

:::note Where's Info.plist?
New projects don't have a separate `Info.plist` file. Its values live in the target's **Info** tab and build settings, and Xcode generates the real `Info.plist` when it builds. You'll add keys there, like permission descriptions (Lesson 27).
:::

## Targets and schemes

- A **target** is one thing Xcode builds: your app, a test bundle, a widget extension. Each target has its own settings and files.
- A **scheme** says what to build, run, test and profile. The scheme menu sits next to the run destination in the toolbar.

Click the blue **project icon** at the top of the navigator, then select the **HelloiOS** target. The tabs across the top are where you'll configure your app.

### General

- **Display Name**: the name under the icon on the Home Screen.
- **Bundle Identifier**: the app's unique ID, e.g. `com.yourname.HelloiOS`.
- **Version** (e.g. `1.0`, shown to users) and **Build** (e.g. `1`, must increase for every upload to App Store Connect).
- **Minimum Deployments**: the deployment target. Set it to **iOS 18.0** for this course.
- **Supported Destinations**: iPhone, iPad, Mac (Designed for iPad), Vision.

### Signing & Capabilities

Your signing team and certificates, plus **capabilities**: special permissions an app needs Apple to grant, like Push Notifications, iCloud, App Groups, Sign in with Apple or HealthKit. Add them with **+ Capability**.

### Info

The keys that go into Info.plist: permission messages, supported orientations, URL types for deep links, background modes, and more.

### Build Settings

Hundreds of compiler and build options. You'll rarely need to change them. Use the search field (e.g. search "Swift Language Version" or "deployment").

## Asset catalogs

`Assets.xcassets` manages images and colours:

- **AppIcon**: drop in a single 1024×1024 image; Xcode generates every size. You can provide separate light, dark and tinted variants.
- **AccentColor**: your app's tint colour for buttons, links and toggles.
- **Image sets**: drag PNG, JPEG, PDF or SVG files in. Use them with `Image("name")`.
- **Color sets**: named colours with separate light and dark values. Use them with `Color("name")`.

Xcode also generates Swift symbols for your assets, so you can write `Image(.pancakes)` and `Color(.brandPrimary)` with autocomplete and compile-time checking.

:::tip Prefer SF Symbols for icons
Apple's [SF Symbols](https://developer.apple.com/sf-symbols/) library has thousands of icons that match the system font, scale with Dynamic Type and support multiple colours. Download the free **SF Symbols** Mac app to browse them, then use `Image(systemName: "heart.fill")`.
:::

## Text and localisation: String Catalogs

Every string you write in SwiftUI, like `Text("Hello")`, is automatically **localisable**. Add a **String Catalog** (**File → New → File from Template → String Catalog**, named `Localizable`), then build. Xcode finds your strings and lists them in the catalog, ready for translation. Add a language with **+** at the bottom of the catalog, and Xcode tracks which strings still need translating.

## Swift Package Manager

To use an open-source library, choose **File → Add Package Dependencies…**, paste the package's Git URL, choose a version rule (usually *Up to Next Major Version*) and add it to your target. Xcode downloads it and keeps the version in `Package.resolved`.

Apple's frameworks cover a huge amount, so you'll need fewer third-party libraries than on other platforms. This course only uses Apple frameworks.

## Building from the command line

`xcodebuild` builds and tests from the Terminal, which is how CI servers do it:

```bash
xcodebuild -scheme HelloiOS -destination 'platform=iOS Simulator,name=iPhone 17' build
xcodebuild -scheme HelloiOS -destination 'platform=iOS Simulator,name=iPhone 17' test
```

(Use the name of any simulator you've installed. `xcrun simctl list devices` lists them.)

## Recap

:::recap
- Navigator (left), editor + canvas (centre), inspectors (right), debug area (bottom).
- The **project** holds **targets** (app, tests, extensions); **schemes** decide what to build and run.
- **General** tab: display name, bundle ID, version/build, deployment target.
- **Info** tab: Info.plist keys. **Signing & Capabilities**: team and special entitlements.
- **Assets.xcassets** for icons, images and colours; **String Catalogs** for translations; **Swift packages** for libraries.
:::

:::exercise
1. Change the **Display Name** to "My First App", run, and check the Home Screen name.
2. Set **Minimum Deployments** to iOS 18.0.
3. In `Assets.xcassets`, set **AccentColor** to a colour you like and see the globe icon change (it uses `.tint`).
4. Add a Color Set called `Brand` with different light and dark values, and use it: `Text("Hello").foregroundStyle(Color("Brand"))`. Toggle dark mode in the preview.
:::

## Check your understanding

```quiz
Q: What's the difference between Version and Build?
- [ ] They're the same thing
- [x] Version (e.g. 1.2) is shown to users; Build (e.g. 42) must increase with every upload
- [ ] Build is shown to users; Version is internal
- [ ] Version is for iOS, Build is for macOS
> App Store Connect rejects an upload whose build number was already used for that version.

Q: Where do you add a capability like Push Notifications?
- [ ] In ContentView.swift
- [ ] In Assets.xcassets
- [x] In the target's Signing & Capabilities tab
- [ ] It's automatic
> Capabilities are entitlements Apple must grant to your signed app.

Q: How do you add a third-party Swift library to your app?
- [ ] Copy its files into your folder
- [x] File → Add Package Dependencies… with the package's Git URL
- [ ] Download a .framework from a website
- [ ] Install it with Homebrew
> Swift Package Manager is built into Xcode.

Q: What's the recommended source for app icons like a heart or a trash can?
- [ ] Screenshots of other apps
- [x] SF Symbols, via `Image(systemName:)`
- [ ] Emoji
- [ ] PNG files at every size
> SF Symbols scale with text, support weights and multicolour rendering, and look native.
```
