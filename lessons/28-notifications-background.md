---
id: notifications-background
title: Notifications & Background Work
part: 6
minutes: 35
summary: Keep users informed and data fresh. Covers local notifications (permission, scheduling, handling taps), how push notifications work, and background app refresh.
---

:::goals
- Asking permission to send notifications
- Scheduling local notifications with time and calendar triggers
- Handling notification taps and showing them while the app is open
- How remote (push) notifications work
- Background refresh with `BGAppRefreshTask` and SwiftUI's `.backgroundTask`
:::

## Local vs. push notifications

- **Local notifications** are scheduled by your app on the device: reminders, timers, "your bread is ready". No server needed.
- **Push (remote) notifications** are sent from your server through Apple's Push Notification service (APNs): new messages, breaking news.

Both show up the same way to the user, and both need the user's **permission**.

## Asking permission

```swift title="NotificationManager.swift"
import Foundation
import UserNotifications

enum NotificationManager {
    /// Returns true if notifications are allowed.
    static func requestPermission() async -> Bool {
        let center = UNUserNotificationCenter.current()
        do {
            return try await center.requestAuthorization(options: [.alert, .sound, .badge])
        } catch {
            return false
        }
    }

    static func isAuthorized() async -> Bool {
        let settings = await UNUserNotificationCenter.current().notificationSettings()
        return settings.authorizationStatus == .authorized
    }
}
```

The system prompt only appears **once**. If the user declines, you can't ask again, only send them to Settings. So ask at a meaningful moment, like when they turn on "Daily recipe reminders", never at first launch.

## Scheduling a local notification

A notification has **content** (title, body, sound) and a **trigger** (when):

```swift title="Reminders.swift"
import Foundation
import UserNotifications

enum Reminders {
    /// Fires once, after a delay.
    static func timer(minutes: Double, recipeName: String) async throws {
        let content = UNMutableNotificationContent()
        content.title = "⏰ Time's up!"
        content.body = "\(recipeName) is ready. Take it out of the oven."
        content.sound = .default
        content.userInfo = ["recipeID": 42]              // data you'll read when it's tapped

        let trigger = UNTimeIntervalNotificationTrigger(timeInterval: minutes * 60, repeats: false)
        let request = UNNotificationRequest(identifier: "timer-\(recipeName)", content: content, trigger: trigger)
        try await UNUserNotificationCenter.current().add(request)
    }

    /// Fires every day at 9:00.
    static func daily() async throws {
        let content = UNMutableNotificationContent()
        content.title = "What's cooking today?"
        content.body = "We picked a new recipe for you. 🍳"
        content.sound = .default

        var time = DateComponents()
        time.hour = 9
        time.minute = 0
        let trigger = UNCalendarNotificationTrigger(dateMatching: time, repeats: true)
        let request = UNNotificationRequest(identifier: "daily-recipe", content: content, trigger: trigger)
        try await UNUserNotificationCenter.current().add(request)   // same identifier replaces the old one
    }

    static func cancelDaily() {
        UNUserNotificationCenter.current().removePendingNotificationRequests(withIdentifiers: ["daily-recipe"])
    }
}
```

Hooking it up to a toggle:

```swift title="ReminderToggle.swift"
import SwiftUI

struct ReminderToggle: View {
    @AppStorage("dailyReminder") private var enabled = false
    @State private var showDenied = false

    var body: some View {
        Toggle("Daily recipe reminder", isOn: $enabled)
            .onChange(of: enabled) { _, isOn in
                Task {
                    if isOn {
                        if await NotificationManager.requestPermission() {
                            try? await Reminders.daily()
                        } else {
                            enabled = false
                            showDenied = true
                        }
                    } else {
                        Reminders.cancelDaily()
                    }
                }
            }
            .alert("Notifications are off", isPresented: $showDenied) {
                Button("OK", role: .cancel) { }
            } message: {
                Text("Turn on notifications for RecipeBox in Settings to get daily ideas.")
            }
    }
}

#Preview { ReminderToggle().padding() }
```

:::tip Testing
Schedule a notification a few seconds ahead, then press [[Cmd]] + [[Shift]] + [[H]] to go to the Home Screen in the Simulator. By default, notifications aren't shown while your app is in the foreground (see below).
:::

## Handling taps, and showing notifications in the foreground

Handle notification events with a `UNUserNotificationCenterDelegate`, set up via an app delegate:

```swift title="AppDelegate.swift" nocheck
import SwiftUI
import UserNotifications

final class AppDelegate: NSObject, UIApplicationDelegate, UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        return true
    }

    // Show banners even while the app is open
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                willPresent notification: UNNotification) async -> UNNotificationPresentationOptions {
        [.banner, .sound]
    }

    // The user tapped a notification
    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                didReceive response: UNNotificationResponse) async {
        let info = response.notification.request.content.userInfo
        if let id = info["recipeID"] as? Int {
            await MainActor.run {
                // e.g. open recipebox://recipe/<id> via your deep link handling
                print("Open recipe \(id)")
            }
        }
    }
}

@main
struct RecipeBoxApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate
    var body: some Scene { WindowGroup { ContentView() } }
}
```

## Push notifications (overview)

Push notifications require the paid Developer Program and a server:

1. Add the **Push Notifications** capability in Signing & Capabilities.
2. Call `UIApplication.shared.registerForRemoteNotifications()` after getting permission.
3. The app delegate receives a **device token** in `application(_:didRegisterForRemoteNotificationsWithDeviceToken:)`. Send it to your server.
4. Your server sends notifications to APNs with that token (services like Firebase Cloud Messaging can do this part for you).

Test pushes in the Simulator by dragging an `.apns` JSON file onto it, or with `xcrun simctl push`.

## Background work

iOS suspends apps shortly after they leave the screen, to protect battery life. Your options for background work:

| Need | Tool |
|---|---|
| Finish a short task when the user leaves (e.g. saving) | It usually completes; for longer work use `beginBackgroundTask` |
| Refresh content occasionally so it's fresh when opened | **Background App Refresh** (`BGAppRefreshTask`) |
| Long processing (e.g. database cleanup) while charging | `BGProcessingTask` |
| Downloads/uploads that continue after the app is suspended | Background `URLSession` |
| Music, navigation, calls | Special background modes |

### Background App Refresh in SwiftUI

1. In **Signing & Capabilities**, add **Background Modes** and tick **Background fetch**.
2. In the **Info** tab, add **Permitted background task scheduler identifiers** (`BGTaskSchedulerPermittedIdentifiers`) containing `com.yourname.recipebox.refresh`.
3. Schedule a request, and handle it with the `.backgroundTask` scene modifier:

```swift title="RefreshApp.swift" nocheck
import SwiftUI
import BackgroundTasks

let refreshTaskID = "com.yourname.recipebox.refresh"

func scheduleAppRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: refreshTaskID)
    request.earliestBeginDate = .now.addingTimeInterval(4 * 60 * 60)   // no sooner than 4 hours
    try? BGTaskScheduler.shared.submit(request)
}

@main
struct RecipeBoxApp: App {
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, phase in
                if phase == .background { scheduleAppRefresh() }
            }
            .backgroundTask(.appRefresh(refreshTaskID)) {
                scheduleAppRefresh()                       // schedule the next one
                await refreshRecipesCache()                // your async work: keep it short
            }
    }
}
```

:::note iOS decides when
Background refresh is **opportunistic**: iOS runs it based on how often the user opens your app, battery and network. It may run in hours, or not at all if the user rarely uses the app or turned Background App Refresh off. Never rely on exact timing; for time-critical alerts, use scheduled local notifications or push.
:::

## Recap

:::recap
- Ask for notification permission at a meaningful moment; the prompt only appears once.
- Schedule local notifications with `UNTimeIntervalNotificationTrigger` or `UNCalendarNotificationTrigger`; reuse identifiers to replace.
- A `UNUserNotificationCenterDelegate` handles taps and foreground presentation.
- Push needs APNs, a device token and a server.
- Background refresh is scheduled with `BGTaskScheduler` and handled with `.backgroundTask`, and iOS decides when it runs.
:::

:::exercise
1. Add a **cooking timer** to a recipe screen: a `Stepper` for minutes and a "Start timer" button that schedules a local notification (asking permission first).
2. Make tapping the notification open that recipe using the deep-link handling from the previous lesson.
:::

## Check your understanding

```quiz
Q: The user denied notification permission. Can your app show the system prompt again?
- [ ] Yes, whenever it wants
- [x] No. You can only direct the user to Settings
- [ ] Yes, after reinstalling the app automatically
- [ ] Only once a day
> The authorization prompt appears once; after that the choice lives in Settings.

Q: Which trigger fires every day at 9:00?
- [ ] UNTimeIntervalNotificationTrigger(timeInterval: 86400, repeats: false)
- [x] UNCalendarNotificationTrigger with hour = 9, minute = 0, repeats: true
- [ ] A Timer in the app
- [ ] BGAppRefreshTask
> Calendar triggers match date components, like hour and minute.

Q: By default, what happens to a notification that arrives while your app is in the foreground?
- [ ] It's shown as usual
- [x] It isn't shown unless your delegate's willPresent returns presentation options
- [ ] The app crashes
- [ ] It's queued until the app closes
> Implement willPresent to show banners in the foreground.

Q: When does a BGAppRefreshTask run?
- [ ] Exactly at earliestBeginDate
- [x] Opportunistically, when iOS decides, based on usage, battery and network
- [ ] Every 15 minutes
- [ ] Only while the app is open
> earliestBeginDate is a minimum, not a schedule.
```
