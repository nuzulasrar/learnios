---
id: system-integration
title: Sharing, Links, Permissions & Photos
part: 6
minutes: 35
summary: Connect your app to the rest of iOS. Covers opening links, the share sheet, deep links into your app, the permission model with usage descriptions, the photo picker, and location.
---

:::goals
- Opening web pages, phone numbers and other apps with `Link` and `openURL`
- Sharing with `ShareLink`
- Deep links: custom URL schemes and `onOpenURL`
- How permissions work: usage descriptions and user prompts
- Picking photos with `PhotosPicker` (no permission needed!)
- Getting the user's location
:::

## Opening URLs

```swift title="LinksDemo.swift"
import SwiftUI

struct LinksDemo: View {
    @Environment(\.openURL) private var openURL

    var body: some View {
        VStack(spacing: 16) {
            Link("Open Swift.org", destination: URL(string: "https://swift.org")!)   // opens Safari

            Button("Call the restaurant") {
                openURL(URL(string: "tel:+15551234567")!)          // Phone app
            }
            Button("Email support") {
                openURL(URL(string: "mailto:help@example.com?subject=RecipeBox")!)
            }
            Button("Directions") {
                openURL(URL(string: "https://maps.apple.com/?q=Pizza")!)  // Apple Maps
            }
        }
    }
}

#Preview { LinksDemo() }
```

Links in `Text` with Markdown are tappable too: `Text("Read our [privacy policy](https://example.com/privacy)")`.

## Sharing: `ShareLink`

`ShareLink` opens the system **share sheet**: Messages, Mail, AirDrop, Notes, other apps and more.

```swift
ShareLink(item: URL(string: "https://dummyjson.com/recipes/1")!) {
    Label("Share recipe", systemImage: "square.and.arrow.up")
}

ShareLink(item: "Try this pancake recipe! 🥞",
          subject: Text("Recipe idea"),
          message: Text("From RecipeBox"))

// Share a rendered image of a view:
ShareLink(item: Image("pancakes"), preview: SharePreview("Pancakes", image: Image("pancakes")))
```

You can share any type that conforms to `Transferable`, which `String`, `URL`, `Image` and `Data` already do.

## Deep links into your app

A **deep link** opens a specific screen of your app from a URL, for example from a notification, a widget, a web page or another app.

1. In the target's **Info** tab, open **URL Types**, click **+**, and set **URL Schemes** to `recipebox`.
2. Handle incoming URLs with `onOpenURL`:

```swift title="DeepLinkApp.swift"
import SwiftUI

struct DeepLinkRoot: View {
    @State private var path: [Int] = []

    var body: some View {
        NavigationStack(path: $path) {
            Text("Home")
                .navigationTitle("RecipeBox")
                .navigationDestination(for: Int.self) { id in
                    Text("Recipe #\(id)").font(.largeTitle)
                }
        }
        .onOpenURL { url in                             // recipebox://recipe/42
            guard url.scheme == "recipebox",
                  url.host() == "recipe",
                  let id = Int(url.lastPathComponent) else { return }
            path = [id]                                 // navigate straight to it
        }
    }
}

#Preview { DeepLinkRoot() }
```

Test it in the Simulator from Terminal:

```bash
xcrun simctl openurl booted "recipebox://recipe/42"
```

(`https://` links that open your app directly are called **Universal Links**. They also require an `apple-app-site-association` file on your website and the Associated Domains capability.)

## How permissions work on iOS

Sensitive data and hardware are protected. To access them your app must:

1. **Declare a usage description** in Info.plist explaining *why*. iOS shows your text in the permission prompt.
2. **Request access** at runtime, ideally when the user starts the feature that needs it.
3. **Handle "denied"** gracefully. The user can also change their mind later in Settings.

| Resource | Info.plist key | Shown as |
|---|---|---|
| Camera | `NSCameraUsageDescription` | Privacy – Camera Usage Description |
| Location | `NSLocationWhenInUseUsageDescription` | Privacy – Location When In Use Usage Description |
| Microphone | `NSMicrophoneUsageDescription` | Privacy – Microphone Usage Description |
| Contacts | `NSContactsUsageDescription` | Privacy – Contacts Usage Description |
| Photo library (full access) | `NSPhotoLibraryUsageDescription` | Privacy – Photo Library Usage Description |
| Face ID | `NSFaceIDUsageDescription` | Privacy – Face ID Usage Description |

Add keys in the target's **Info** tab: hover over any row, click **+**, and type "Privacy". Write a clear, specific reason:

```plist
<key>NSCameraUsageDescription</key>
<string>RecipeBox uses the camera to scan recipe cards.</string>
```

:::danger Missing usage descriptions crash the app
If your app accesses a protected resource without the matching usage description, iOS **terminates it immediately**, and App Review rejects apps with vague descriptions like "Needs camera". Explain the benefit to the user.
:::

If the user denies access, send them to your app's page in Settings:

```swift
Button("Open Settings") {
    openURL(URL(string: UIApplication.openSettingsURLString)!)
}
```

## Picking photos: `PhotosPicker`

The system photo picker runs **outside your app**: the user chooses photos and your app only receives those. **No permission prompt and no usage description needed.**

```swift title="PhotoPickerDemo.swift"
import SwiftUI
import PhotosUI

struct PhotoPickerDemo: View {
    @State private var selection: PhotosPickerItem?
    @State private var image: Image?

    var body: some View {
        VStack(spacing: 20) {
            if let image {
                image
                    .resizable()
                    .scaledToFill()
                    .frame(width: 240, height: 240)
                    .clipShape(.rect(cornerRadius: 20))
            } else {
                ContentUnavailableView("No photo", systemImage: "photo")
                    .frame(height: 240)
            }

            PhotosPicker("Choose a photo", selection: $selection, matching: .images)
                .buttonStyle(.borderedProminent)
        }
        .task(id: selection) {                                      // runs when the selection changes
            guard let selection else { return }
            if let data = try? await selection.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                image = Image(uiImage: uiImage)
            }
        }
    }
}

#Preview { PhotoPickerDemo() }
```

Use `PhotosPicker(selection: $items, maxSelectionCount: 5, matching: .images)` with an array to pick several. To save the photo with a recipe, store the `Data` (e.g. in a SwiftData property marked `@Attribute(.externalStorage)`).

## Location

Location needs permission. Modern APIs make it concise:

```swift title="LocationDemo.swift" nocheck
import SwiftUI
import CoreLocation

@MainActor
@Observable
final class LocationModel {
    private(set) var location: CLLocation?
    private(set) var denied = false
    private let manager = CLLocationManager()

    func start() async {
        manager.requestWhenInUseAuthorization()            // shows the prompt (once)
        do {
            for try await update in CLLocationUpdate.liveUpdates() {
                if update.authorizationDenied { denied = true; break }
                if let loc = update.location {
                    location = loc
                    break                                   // we only need one fix
                }
            }
        } catch {
            denied = true
        }
    }
}

struct LocationDemo: View {
    @State private var model = LocationModel()

    var body: some View {
        VStack {
            if let loc = model.location {
                Text("You're at \(loc.coordinate.latitude, specifier: "%.3f"), \(loc.coordinate.longitude, specifier: "%.3f")")
            } else if model.denied {
                Text("Location access is off. Enable it in Settings to find restaurants near you.")
            } else {
                Button("Find restaurants near me") { Task { await model.start() } }
            }
        }
        .padding()
    }
}
```

Remember to add `NSLocationWhenInUseUsageDescription`. In the Simulator, choose a location via **Features → Location**. For showing maps, SwiftUI has a `Map` view in **MapKit** with markers, annotations and more.

## Recap

:::recap
- `Link` and `openURL` open websites, phone, mail and maps; `ShareLink` opens the share sheet.
- Deep links: register a URL scheme, handle it with `onOpenURL`, and drive your navigation path.
- Protected resources need a **usage description** plus a runtime request; handle denial gracefully.
- `PhotosPicker` needs no permission at all.
:::

:::exercise
1. Add a **Share** button and a **Call** button to a detail screen.
2. Register the `recipebox` URL scheme and make `recipebox://recipe/7` open the detail for recipe 7. Test it with `xcrun simctl openurl`.
3. Add a profile avatar that opens a `PhotosPicker` and shows the chosen image in a circle.
:::

## Check your understanding

```quiz
Q: What happens if your app accesses the camera without `NSCameraUsageDescription` in Info.plist?
- [ ] The permission prompt shows a default message
- [x] iOS terminates the app
- [ ] Nothing, it works
- [ ] The camera shows a black screen
> Usage descriptions are mandatory for protected resources.

Q: Which permission does `PhotosPicker` need?
- [ ] Full photo library access
- [ ] Camera access
- [x] None, because the picker runs outside your app and only returns what the user chooses
- [ ] Files access
> The out-of-process picker is private by design.

Q: How do you handle a URL like `recipebox://recipe/42` in SwiftUI?
- [ ] Info.plist handles it automatically
- [x] Register the URL scheme, then use `.onOpenURL` to parse it and navigate
- [ ] Use a WebView
- [ ] It's not possible
> onOpenURL receives the URL; your code decides where to go.

Q: When is the best time to request location permission?
- [ ] At first launch, together with every other permission
- [x] When the user starts a feature that needs it, so the reason is obvious
- [ ] Never, just read it
- [ ] After the app is rated
> Contextual requests get far more "Allow" taps.
```
