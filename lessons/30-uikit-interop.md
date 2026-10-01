---
id: uikit-interop
title: UIKit & Interop
part: 6
minutes: 30
summary: Many apps and libraries still use UIKit. Learn enough to read UIKit code, wrap UIKit views and controllers for SwiftUI, and host SwiftUI inside UIKit apps.
---

:::goals
- What UIKit is and how it differs from SwiftUI
- View controllers, views and Auto Layout in brief
- `UIViewRepresentable` to wrap a UIKit view for SwiftUI
- `UIViewControllerRepresentable` with a `Coordinator` for delegates
- `UIHostingController` to put SwiftUI inside UIKit
:::

## Why learn UIKit at all?

SwiftUI is the future, and everything new in this course uses it. But **UIKit** has powered iOS apps since 2008, and:

- Most established apps are large UIKit codebases migrating gradually, so you'll meet UIKit at work.
- A few features have richer UIKit APIs, and some third-party SDKs only provide UIKit views.
- You can mix both freely, screen by screen or view by view.

## UIKit in a nutshell

UIKit is **imperative** and **class-based**:

- A **`UIViewController`** manages one screen: its views, lifecycle (`viewDidLoad`, `viewWillAppear`…) and user interactions.
- **`UIView`** subclasses (`UILabel`, `UIButton`, `UIImageView`, `UITableView`…) draw things on screen.
- **Auto Layout** positions views with constraints.
- **Delegates** and **target–action** deliver events.

```swift title="CounterViewController.swift (UIKit)" nocheck
import UIKit

final class CounterViewController: UIViewController {
    private var count = 0
    private let label = UILabel()
    private let button = UIButton(configuration: .filled())

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground

        label.font = .preferredFont(forTextStyle: .title1)
        label.text = "Tapped 0 times"
        button.configuration?.title = "Tap me"
        button.addAction(UIAction { [weak self] _ in self?.tapped() }, for: .touchUpInside)

        let stack = UIStackView(arrangedSubviews: [label, button])
        stack.axis = .vertical
        stack.spacing = 16
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            stack.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }

    private func tapped() {
        count += 1
        label.text = "Tapped \(count) times"      // YOU update the label manually
    }
}
```

Compare this with the SwiftUI counter from Lesson 12: here you create views, lay them out with constraints, and update the label by hand whenever the count changes.

| UIKit | SwiftUI equivalent |
|---|---|
| `UIViewController` | A `View` (screen) |
| `UILabel`, `UIButton`, `UIImageView` | `Text`, `Button`, `Image` |
| `UIStackView` + Auto Layout | `VStack` / `HStack` + modifiers |
| `UITableView` / `UICollectionView` | `List` / `LazyVGrid` |
| `UINavigationController` | `NavigationStack` |
| `UITabBarController` | `TabView` |
| Delegates / target–action | Closures and bindings |
| Storyboards / XIBs | Code + previews |

:::note `[weak self]`
UIKit code uses closures that capture `self` a lot. `[weak self]` prevents a **retain cycle** (two objects keeping each other alive forever, leaking memory). You'll see it constantly in UIKit and in code with long-lived closures.
:::

## UIKit inside SwiftUI: `UIViewRepresentable`

To use a UIKit view in SwiftUI, wrap it in a struct conforming to `UIViewRepresentable`:

```swift title="ActivitySpinner.swift" nocheck
import SwiftUI
import UIKit

struct ActivitySpinner: UIViewRepresentable {
    var isAnimating: Bool
    var color: UIColor = .systemOrange

    func makeUIView(context: Context) -> UIActivityIndicatorView {    // create once
        let view = UIActivityIndicatorView(style: .large)
        view.color = color
        return view
    }

    func updateUIView(_ view: UIActivityIndicatorView, context: Context) {   // on every SwiftUI update
        isAnimating ? view.startAnimating() : view.stopAnimating()
    }
}

struct SpinnerDemo: View {
    @State private var loading = true
    var body: some View {
        VStack {
            ActivitySpinner(isAnimating: loading)
            Toggle("Loading", isOn: $loading).padding()
        }
    }
}
```

- `makeUIView` creates the UIKit view **once**.
- `updateUIView` runs whenever the SwiftUI state it depends on changes. Push new values into the view here.

## Wrapping a view controller with a Coordinator

Many UIKit components report events through **delegates**. A **Coordinator** is an object you provide to act as the delegate and pass results back to SwiftUI. Here's the system camera picker, useful for taking a new photo:

```swift title="CameraPicker.swift" nocheck
import SwiftUI
import UIKit

struct CameraPicker: UIViewControllerRepresentable {
    @Binding var image: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator          // the coordinator receives delegate calls
        return picker
    }

    func updateUIViewController(_ controller: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(parent: self) }

    final class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraPicker
        init(parent: CameraPicker) { self.parent = parent }

        func imagePickerController(_ picker: UIImagePickerController,
                                   didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
            parent.image = info[.originalImage] as? UIImage       // send the result back to SwiftUI
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

// Usage (remember NSCameraUsageDescription, and use a real device, since the Simulator has no camera):
// .fullScreenCover(isPresented: $showCamera) { CameraPicker(image: $photo).ignoresSafeArea() }
```

## SwiftUI inside UIKit: `UIHostingController`

Going the other way is one line. That's how teams add new SwiftUI screens to existing UIKit apps:

```swift nocheck
let settings = UIHostingController(rootView: SettingsView())
navigationController?.pushViewController(settings, animated: true)
```

You can even use SwiftUI for individual table or collection view cells with `UIHostingConfiguration`.

## Recap

:::recap
- UIKit is imperative and class-based: view controllers, views, Auto Layout, delegates.
- `UIViewRepresentable` / `UIViewControllerRepresentable` wrap UIKit for SwiftUI: create in `make…`, update in `update…`.
- A **Coordinator** handles delegate callbacks and passes results back through bindings.
- `UIHostingController` puts SwiftUI into UIKit apps.
:::

:::exercise
1. Wrap `UITextView` in a `UIViewRepresentable` with `@Binding var text: String`. Use a Coordinator as the `UITextViewDelegate` to write changes back to the binding in `textViewDidChange`.
2. Use your wrapper in a SwiftUI form and show the character count below it.
:::

## Check your understanding

```quiz
Q: How do you use a UIKit view inside SwiftUI?
- [ ] Add it directly to a VStack
- [x] Wrap it in a type conforming to UIViewRepresentable
- [ ] Use UIHostingController
- [ ] It's not possible
> Representables bridge UIKit into SwiftUI.

Q: What's the role of a Coordinator in a representable?
- [ ] It draws the view
- [x] It acts as the delegate/target for UIKit events and passes results back to SwiftUI
- [ ] It replaces the view controller
- [ ] It manages navigation
> Coordinators connect UIKit's delegate pattern to SwiftUI's data flow.

Q: How do you show a SwiftUI view inside an existing UIKit app?
- [x] Wrap it in UIHostingController
- [ ] Wrap it in UIViewRepresentable
- [ ] Convert it to a storyboard
- [ ] You must rewrite the app
> UIHostingController is a UIViewController that hosts SwiftUI content.

Q: Why write `[weak self]` in UIKit closures?
- [ ] To make them faster
- [x] To avoid retain cycles that leak memory
- [ ] It's required syntax
- [ ] To run them on the main thread
> A strong capture of self in a closure that self owns keeps both alive forever.
```
