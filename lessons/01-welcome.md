---
id: welcome
title: Welcome & What Is iOS?
part: 1
minutes: 15
summary: How to get the most out of this course, what iOS actually is, how an app gets onto an iPhone, and the modern Apple toolkit you're about to learn.
---

:::goals
- How to use this course so you actually *learn*, not just read
- What iOS is and how an app runs on an iPhone
- What you need to get started (and what you don't)
- Why we use Swift and SwiftUI
- iOS versions and deployment targets
:::

## Welcome!

By the end of this course you'll be able to design, build, test and ship a complete native iOS app. You don't need any previous iOS experience. It helps if you've programmed a little before (in any language), but Part 2 teaches Swift from scratch, so a complete beginner can follow along too.

The course has **8 parts**. The first seven teach one topic at a time. The eighth is a **capstone project**: you'll build *RecipeBox*, a real app that loads recipes from the internet, works offline, saves favourites with SwiftData, supports dark mode, and is ready for TestFlight.

### How to learn effectively

:::tip The golden rule: type the code yourself
Don't copy and paste. Typing code forces your brain to process every line, and the mistakes you make (and fix) are where most of the learning happens.
:::

1. **Read the lesson once** to get the big picture.
2. **Code along** in Xcode (from Lesson 2 onwards). For pure-Swift examples, press **▶ Try online**: it copies the code and opens an online Swift playground. You can also use an Xcode Playground (**File → New → Playground**).
3. **Do the practice tasks.** Try each one before you open the solution. Being stuck for 10 minutes is normal and useful.
4. **Take the quiz** at the end of each lesson, then click **Mark as complete**. Your progress is saved in this browser.
5. **Experiment.** Change values, break things, read the error messages. That's how professionals learn too.

## What is iOS?

**iOS** is the operating system that runs on the iPhone. Its close siblings, iPadOS, macOS, watchOS, tvOS and visionOS, share most of the same frameworks, so what you learn here carries over to every Apple platform.

Like any OS, it's built in layers. Your app sits at the top and uses the layers below through Apple's frameworks:

<div class="diagram">
<div class="stack">
<div class="hl">Your app<small>Swift code + assets (images, colours, strings)</small></div>
<div>UI frameworks<small>SwiftUI (modern) and UIKit (classic)</small></div>
<div>App services<small>Foundation, SwiftData, MapKit, WidgetKit, StoreKit, HealthKit, …</small></div>
<div>Core OS & media<small>Networking, security, graphics (Metal), audio, camera</small></div>
<div>Darwin kernel<small>Memory, processes, drivers, power management</small></div>
</div>
<div class="diagram-caption">The iOS stack. You work almost entirely in the top three layers.</div>
</div>

## What you need

- **A Mac.** Xcode, Apple's development app, only runs on macOS. Xcode 27 needs macOS Tahoe 26.6 or later. An Apple silicon Mac (M1 or newer) is strongly recommended.
- **Xcode** (free, from the Mac App Store). It includes everything: editor, compiler, iPhone Simulator, design previews, debugger and profiler.
- **An Apple Account** (free). It's enough to run apps on your *own* iPhone.
- **Optional: an iPhone.** The Simulator covers most learning. A real device is best for testing the camera, performance and how the app feels.
- **Later: the Apple Developer Program** (paid yearly membership). Only needed to publish on the App Store or share builds through TestFlight. You don't need it for this course until Lesson 33.

## How an app gets onto an iPhone

When you press **Run** in Xcode, a lot happens automatically:

<div class="diagram">
<div class="flow">
<div>Swift source<small>.swift files</small></div>
<div class="arrow">→</div>
<div>Swift compiler<small>native ARM machine code</small></div>
<div class="arrow">→</div>
<div>App bundle<small>.app = code + assets + Info.plist</small></div>
<div class="arrow">→</div>
<div>Code signing<small>proves who built it</small></div>
<div class="arrow">→</div>
<div class="hl">Installed<small>Simulator or iPhone</small></div>
</div>
<div class="diagram-caption">Swift compiles straight to fast native code. There's no virtual machine.</div>
</div>

- The **app bundle** (`.app`) is a folder containing your compiled code, images, the app icon and **Info.plist**, the app's description of itself: its name, version and permission messages.
- **Code signing** is mandatory. iPhones only run apps signed with a certificate Apple trusts. Xcode handles this for you with *automatic signing*.
- For the App Store you upload an **archive** to App Store Connect; Apple reviews it, then delivers optimised versions to each device.

Each app runs in its own **sandbox**: a private folder for its files, no access to other apps' data, and no access to sensitive things (camera, location, photos, contacts) without the user's permission. iOS shows the permission prompts; your job is to explain *why* you need access.

## Native vs. cross-platform

"Native" iOS development means building with Apple's own tools: **Swift**, **SwiftUI** and **Xcode**. Alternatives include Flutter, React Native and Kotlin Multiplatform. Here's why native is a great place to start:

- **Day-one access** to every new Apple API, from widgets and Live Activities to Apple Intelligence features.
- **The best performance** and the most "at home" look and feel, which iPhone users notice.
- Most iOS jobs require it, and it makes you better at cross-platform tools later.
- Swift and SwiftUI also build apps for iPad, Mac, Apple Watch, Apple TV and Vision Pro.

## The modern Apple toolkit

| Tool | What it's for | Where you'll learn it |
|---|---|---|
| **Swift** | The programming language: fast, safe and expressive | Part 2 |
| **Swift Concurrency** | `async`/`await` and actors for slow work without freezing the UI | Lesson 11 |
| **SwiftUI** | Building UI declaratively in Swift code | Part 3 |
| **Observation** | `@Observable` models that SwiftUI tracks automatically | Lesson 21 |
| **NavigationStack & TabView** | Moving between screens | Lesson 20 |
| **URLSession & Codable** | Networking and JSON | Lesson 23 |
| **SwiftData** | Persisting your app's data | Lesson 24 |
| **Swift Testing** | Modern unit tests | Lesson 31 |
| **Xcode, Instruments, TestFlight** | Build, debug, profile and ship | Parts 1 & 7 |

## iOS versions & deployment targets

Apple releases a major iOS version every September. In 2025 Apple switched to **year-based version numbers**, jumping from iOS 18 to **iOS 26**, so all its platforms share one number.

| Version | Released | Highlights for developers |
|---|---|---|
| iOS 27 | Sept 2026 | The current release (Xcode 27, Swift 6.4) |
| iOS 26 | Sept 2025 | *Liquid Glass* design, the year-based naming |
| iOS 18 | Sept 2024 | New `Tab` API, Swift 6, Swift Testing |
| iOS 17 | Sept 2023 | Observation (`@Observable`) and SwiftData |
| iOS 16 | Sept 2022 | `NavigationStack` |

Your project's **deployment target** (minimum iOS version) decides the oldest iOS your app runs on. iPhone users update quickly, so most apps support **the current version and one or two before it**. This course uses APIs from iOS 18 and later, and a deployment target of **iOS 18** is a good choice.

:::analogy
The deployment target is the oldest phone software you promise to support. The **SDK** is the newest dictionary of APIs you're allowed to look words up in. Xcode always builds with the latest SDK, and you guard newer APIs with `if #available(iOS 27, *)` when you need them.
:::

## Your roadmap

1. **Getting Started**: install Xcode and run an app (you're here).
2. **Swift Essentials**: the language, from values to concurrency.
3. **Building UI with SwiftUI**: layout, controls, state, lists, styling, animation.
4. **App Architecture**: lifecycle, navigation, Observation & MVVM, dependency injection.
5. **Data & Networking**: URLSession, Codable, SwiftData, settings, offline caching.
6. **The iOS Platform**: sharing, permissions, notifications, widgets, UIKit.
7. **Quality & Shipping**: testing, debugging, App Store and TestFlight.
8. **Capstone: RecipeBox**: build a complete app from scratch.

## Check your understanding

```quiz
Q: Which of these do you need to build iOS apps with Xcode?
- [ ] A Windows PC with Visual Studio
- [x] A Mac
- [ ] A paid Apple Developer Program membership from day one
- [ ] An iPhone
> Xcode runs only on macOS. A free Apple Account is enough to run apps on your own device; the paid program is only needed for the App Store and TestFlight.

Q: What is Info.plist?
- [ ] The compiled Swift code
- [x] A file in the app bundle describing the app: name, version, permission messages and more
- [ ] A list of the app's users
- [ ] The Xcode project file
> Info.plist is the app's self-description that iOS reads before running any code.

Q: Your app's deployment target is iOS 18. What does that mean?
- [ ] It only runs on iOS 18
- [x] It can be installed on iOS 18 and newer
- [ ] It's built with the iOS 18 SDK
- [ ] It needs an iPhone released in 2024
> The deployment target is the minimum supported version. You always build with the latest SDK.

Q: Why does an app need the user's permission to use the camera?
- [ ] Because Swift requires it
- [ ] Because the camera is slow
- [x] Because each app runs in a sandbox and can't access sensitive resources without consent
- [ ] Because Xcode blocks camera code by default
> iOS protects privacy-sensitive resources and asks the user, showing your explanation from Info.plist.
```
