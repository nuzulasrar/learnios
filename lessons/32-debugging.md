---
id: debugging
title: Debugging & Instruments
part: 7
minutes: 30
summary: Find and fix bugs like a professional. Covers reading crash messages, logging with Logger, breakpoints and LLDB, the view and memory graph debuggers, and profiling with Instruments.
---

:::goals
- Reading crashes and runtime errors
- `print` vs. `Logger` for logging
- Breakpoints, stepping and LLDB commands
- The view hierarchy debugger and memory graph (retain cycles)
- Profiling with Instruments: Time Profiler, SwiftUI, Leaks
- Common crashes and their fixes
:::

## When the app crashes

When a crash happens while running from Xcode, the debugger **pauses at the line that crashed** and the console shows a message. The most common ones:

```text
Fatal error: Unexpectedly found nil while unwrapping an Optional value
Fatal error: Index out of range
Thread 1: EXC_BAD_ACCESS
```

1. Read the **message** in the console (and the red banner in the editor).
2. Look at the **highlighted line** and the **call stack** in the Debug navigator ([[Cmd]] + [[7]]) to see how execution got there. Frames in *your* code are shown in bold.
3. Inspect variables in the **Variables view** at the bottom-left.

:::tip Search smartly
Copy the error message (without your own variable names) into a search engine or the Apple Developer Forums. Someone has almost always hit it before.
:::

## Logging: `print` and `Logger`

`print` is fine for quick experiments. For real apps, use **`Logger`** from the unified logging system. It's fast, has levels, can be filtered in Xcode's console, and hides private data in production logs:

```swift title="Log.swift"
import OSLog

extension Logger {
    static let network = Logger(subsystem: "com.yourname.RecipeBox", category: "network")
    static let storage = Logger(subsystem: "com.yourname.RecipeBox", category: "storage")
}

func example(count: Int, email: String) {
    Logger.network.debug("Fetching recipes…")
    Logger.network.info("Loaded \(count) recipes")
    Logger.network.error("Request failed for \(email, privacy: .private)")   // redacted in production logs
}
```

In Xcode's console you can filter by **category**, **level** or text, and click a log line to jump to the code that produced it.

## Breakpoints

1. Click the **gutter** (line numbers) to add a breakpoint, shown as a blue arrow.
2. Run the app. When that line is about to execute, the app **pauses**.
3. Inspect variables, hover over them in the editor, then use the debug bar:

| Action | Shortcut |
|---|---|
| Continue | [[Ctrl]] + [[Cmd]] + [[Y]] |
| Step over (next line) | [[F6]] |
| Step into (enter the function) | [[F7]] |
| Step out (finish the function) | [[F8]] |
| Enable/disable all breakpoints | [[Cmd]] + [[Y]] |

Right-click a breakpoint → **Edit Breakpoint** to add a **condition** (`recipe.id == 42`), or an **action** that logs a message and automatically continues. That's logging without changing code. In the Breakpoint navigator ([[Cmd]] + [[8]]), click **+ → Swift Error Breakpoint** to pause whenever any error is thrown.

### LLDB in the console

While paused, the console accepts debugger commands:

```text
(lldb) po recipe                       // print an object's description
(lldb) p recipes.count                 // evaluate an expression
(lldb) po viewModel.state
(lldb) expr isLoading = false          // change a value while paused
(lldb) bt                              // backtrace: the call stack
```

## Debugging SwiftUI

- `let _ = Self._printChanges()` inside `body` prints **why** a view re-rendered (debug builds only). It's great for spotting unnecessary updates.
- In the canvas, right-click a preview and choose **Debug Preview** to hit breakpoints in previews.
- **Debug View Hierarchy** (the stacked-rectangles button in the debug bar) shows an exploded 3D view of your UI, with each view's frame and properties. Use it for "why is this invisible / clipped / in the wrong place?"

## Memory: the memory graph and retain cycles

Swift uses **Automatic Reference Counting (ARC)**: a class instance is freed when nothing references it any more. A **retain cycle** (A holds B, B holds A) means neither is ever freed, which is a memory leak.

```swift nocheck
final class TimerModel {
    var timer: Timer?
    var ticks = 0

    func start() {
        // ❌ The timer's closure captures self strongly, and self holds the timer → leak
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
            self.ticks += 1
        }
        // ✅ Capture self weakly instead
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.ticks += 1
        }
    }
}
```

To hunt leaks, click **Debug Memory Graph** in the debug bar (three connected circles). It lists every live object; purple ⚠️ icons flag likely leaks, and you can see who is holding each object.

## Instruments: profiling performance

**Product → Profile** ([[Cmd]] + [[I]]) builds a release-optimised app and opens **Instruments**. Useful templates:

- **Time Profiler**: where the CPU spends time. Find the slow function that makes scrolling janky.
- **SwiftUI**: which views update, how often, and how long `body` takes.
- **Leaks / Allocations**: memory leaks and growth.
- **Network**: requests and their timing.
- **Hangs**: when the main thread was blocked.

:::warning Measure release builds on a real device
Debug builds are unoptimised, and the Simulator uses your Mac's powerful CPU. Performance conclusions should come from **Profile** builds on a real (ideally older) iPhone.
:::

## Common crashes and fixes

| Crash | Usual cause | Fix |
|---|---|---|
| *Unexpectedly found nil while unwrapping* | `!` on a nil optional | Use `if let` / `guard let` / `??` |
| *Index out of range* | `array[i]` with a bad index | Check `indices.contains(i)`, use `first`/`last` |
| *This application has crashed because it attempted to access privacy-sensitive data* | Missing usage description | Add the `NS…UsageDescription` key |
| *No Observable object of type … found* | Missing `.environment(...)` | Provide the object higher up (and in previews) |
| *Publishing changes from background threads* / main actor warnings | Updating UI state off the main actor | Make the view model `@MainActor` |
| `DecodingError.keyNotFound` | JSON doesn't match your model | Make the property optional or fix `CodingKeys` |
| Hang / frozen UI | Heavy work on the main thread | Move it into async code / a background actor |

## A debugging mindset

1. **Reproduce** the bug reliably. Write down the steps.
2. **Read** the error message, all of it.
3. **Form a hypothesis** ("the list is empty because the search filter is wrong").
4. **Test it** with a breakpoint, `po`, a log, or the view debugger.
5. **Fix it**, then add a **test** so it never returns.

:::exercise
1. Put a breakpoint inside a view model's `load()` method, trigger it, and use `po` to inspect its state. Add a condition so it only stops when there's an error.
2. Deliberately crash with `let x: Int? = nil; print(x!)` in a button action, read the crash, then fix it properly.
3. Open **Debug View Hierarchy** on one of your screens and find a view's frame.
4. Add `let _ = Self._printChanges()` to a view's body and watch the console as you interact.
:::

## Check your understanding

```quiz
Q: What's the advantage of Logger over print?
- [ ] It prints in colour
- [x] Levels, categories, filtering in the console, low overhead, and privacy redaction in production
- [ ] It sends logs to a server
- [ ] print doesn't work in iOS apps
> Logger is the unified logging system designed for real apps.

Q: What does `po viewModel.state` do in the debugger console?
- [ ] Changes the state
- [x] Prints a description of the value while the app is paused
- [ ] Restarts the app
- [ ] Posts a notification
> po (print object) evaluates and describes an expression.

Q: A closure stored by an object references `self` strongly. What problem can this cause?
- [ ] A crash on launch
- [x] A retain cycle, where the object is never freed (a memory leak)
- [ ] Slower compilation
- [ ] Nothing
> Use [weak self] to break the cycle.

Q: Why should performance be measured with Product → Profile on a real device?
- [ ] Instruments only works on devices
- [x] Debug builds and the Simulator don't reflect real-world performance
- [ ] It's required for the App Store
- [ ] Profile builds include more logs
> Optimised builds on real hardware show what users actually experience.
```
