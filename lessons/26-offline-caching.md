---
id: offline-caching
title: Offline-First & Caching
part: 5
minutes: 30
summary: Make your app feel instant and work without internet. Show cached data immediately, refresh in the background, detect connectivity, and handle stale data gracefully.
---

:::goals
- What "offline-first" means and why users notice it
- A disk cache for API responses using Codable and files
- The "show cached, then refresh" pattern in a view model
- Detecting connectivity with `NWPathMonitor`
- Caching images with `URLCache`
:::

## The problem with network-only screens

A network-only screen starts with a spinner every time and shows an error whenever the connection drops. Real users are on trains, in lifts and on weak mobile data. Compare:

- **Network-only:** open → spinner → (maybe) data → go offline → error screen.
- **Offline-first:** open → **cached data instantly** → fresh data quietly replaces it → go offline → still usable, with a small "offline" note.

## The pattern

<div class="diagram"><div class="flow">
<div class="hl">1. Load cache<small>instant</small></div><div class="arrow">→</div>
<div>2. Show it</div><div class="arrow">→</div>
<div class="hl">3. Fetch network</div><div class="arrow">→</div>
<div>4. Save to cache + show</div>
</div><div class="diagram-caption">If step 3 fails, the user still has step 2's data, plus a gentle message.</div></div>

## A simple disk cache

Store any `Codable` value as a JSON file in the Caches folder, with the time it was saved:

```swift title="DiskCache.swift"
import Foundation

actor DiskCache {
    struct Entry<Value: Codable>: Codable {
        let savedAt: Date
        let value: Value
    }

    private let folder: URL

    init(name: String = "api-cache") {
        folder = URL.cachesDirectory.appending(path: name)
        try? FileManager.default.createDirectory(at: folder, withIntermediateDirectories: true)
    }

    func save<Value: Codable>(_ value: Value, for key: String) throws {
        let data = try JSONEncoder().encode(Entry(savedAt: .now, value: value))
        try data.write(to: folder.appending(path: "\(key).json"), options: .atomic)
    }

    func load<Value: Codable>(_ type: Value.Type, for key: String) -> Entry<Value>? {
        guard let data = try? Data(contentsOf: folder.appending(path: "\(key).json")) else { return nil }
        return try? JSONDecoder().decode(Entry<Value>.self, from: data)
    }
}
```

It's an `actor`, so reads and writes from different tasks can never clash.

## A caching repository

Combine the network service and the cache behind one type:

```swift title="QuoteRepository.swift"
import Foundation

struct Quote: Codable, Identifiable, Hashable, Sendable {
    let id: Int
    let quote: String
    let author: String
}

protocol QuoteAPI: Sendable {
    func fetchQuotes() async throws -> [Quote]
}

struct LiveQuoteAPI: QuoteAPI {
    func fetchQuotes() async throws -> [Quote] {
        let url = URL(string: "https://dummyjson.com/quotes?limit=50")!
        let (data, response) = try await URLSession.shared.data(from: url)
        guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw URLError(.badServerResponse) }
        struct Page: Decodable { let quotes: [Quote] }
        return try JSONDecoder().decode(Page.self, from: data).quotes
    }
}

struct QuoteRepository: Sendable {
    let api: any QuoteAPI
    let cache: DiskCache
    private let key = "quotes"

    /// Cached quotes and when they were saved, if any.
    func cached() async -> (quotes: [Quote], savedAt: Date)? {
        guard let entry = await cache.load([Quote].self, for: key) else { return nil }
        return (entry.value, entry.savedAt)
    }

    /// Fetch fresh quotes from the network and update the cache.
    func refresh() async throws -> [Quote] {
        let fresh = try await api.fetchQuotes()
        try? await cache.save(fresh, for: key)
        return fresh
    }
}
```

## The view model

```swift title="QuotesViewModel.swift"
import Foundation
import Observation

@MainActor
@Observable
final class OfflineQuotesViewModel {
    private(set) var quotes: [Quote] = []
    private(set) var lastUpdated: Date?
    private(set) var isRefreshing = false
    private(set) var errorMessage: String?

    private let repository: QuoteRepository

    init(repository: QuoteRepository) {
        self.repository = repository
    }

    func load() async {
        // 1–2. Show cached data immediately
        if quotes.isEmpty, let cached = await repository.cached() {
            quotes = cached.quotes
            lastUpdated = cached.savedAt
        }
        // 3–4. Refresh from the network
        await refresh()
    }

    func refresh() async {
        isRefreshing = true
        defer { isRefreshing = false }
        do {
            quotes = try await repository.refresh()
            lastUpdated = .now
            errorMessage = nil
        } catch is CancellationError {
            // ignore
        } catch {
            errorMessage = quotes.isEmpty
                ? "You're offline and there's nothing saved yet."
                : "You're offline. Showing saved quotes."
        }
    }
}
```

## The view

```swift title="OfflineQuotesView.swift"
import SwiftUI

struct OfflineQuotesView: View {
    @State private var viewModel: OfflineQuotesViewModel

    init(repository: QuoteRepository) {
        _viewModel = State(initialValue: OfflineQuotesViewModel(repository: repository))
    }

    var body: some View {
        NavigationStack {
            List {
                if let message = viewModel.errorMessage, !viewModel.quotes.isEmpty {
                    Label(message, systemImage: "wifi.slash")
                        .font(.footnote)
                        .foregroundStyle(.secondary)
                }
                ForEach(viewModel.quotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text("“\(quote.quote)”")
                        Text(quote.author).font(.caption).foregroundStyle(.secondary)
                    }
                }
            }
            .overlay {
                if viewModel.quotes.isEmpty {
                    if let message = viewModel.errorMessage {
                        ContentUnavailableView("No quotes", systemImage: "wifi.slash", description: Text(message))
                    } else {
                        ProgressView()
                    }
                }
            }
            .refreshable { await viewModel.refresh() }
            .task { await viewModel.load() }
            .navigationTitle("Quotes")
            .toolbar {
                if let updated = viewModel.lastUpdated {
                    ToolbarItem(placement: .status) {
                        Text("Updated \(updated, style: .relative) ago").font(.caption)
                    }
                }
            }
        }
    }
}

#Preview {
    OfflineQuotesView(repository: QuoteRepository(api: LiveQuoteAPI(), cache: DiskCache()))
}
```

Test it: run the app online once, then turn off Wi-Fi on your Mac (the Simulator uses its connection) and relaunch. Quotes appear instantly with the offline note, and pull-to-refresh fails gently instead of wiping the screen. 🎉

## Detecting connectivity

`NWPathMonitor` from the Network framework reports connectivity changes. Use it to show an offline banner, or to retry automatically when the connection returns:

```swift title="NetworkMonitor.swift"
import Foundation
import Network
import Observation

@MainActor
@Observable
final class NetworkMonitor {
    private(set) var isConnected = true
    private let monitor = NWPathMonitor()

    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            let connected = path.status == .satisfied
            Task { @MainActor in self?.isConnected = connected }
        }
        monitor.start(queue: DispatchQueue(label: "NetworkMonitor"))
    }

    deinit { monitor.cancel() }
}
```

:::tip Don't pre-check the network
Don't use the monitor to *prevent* requests ("no connection, so don't try"). Connectivity information can be wrong or stale. Always attempt the request and handle failure; use the monitor for UI hints and automatic retries.
:::

## Caching images and HTTP responses

`URLSession` and `AsyncImage` use a shared **`URLCache`** that respects the server's cache headers. You can make it bigger at app launch so images stay cached longer:

```swift
URLCache.shared = URLCache(memoryCapacity: 50_000_000,       // 50 MB in memory
                           diskCapacity: 300_000_000)         // 300 MB on disk
```

For offline-first *writes* (e.g. favouriting while offline), save the change locally first, which SwiftData makes easy, then sync to the server when possible.

## Recap

:::recap
- Offline-first: show cached data instantly, refresh in the background, keep data on failure.
- A small `actor` disk cache + Codable stores API responses with a timestamp.
- The repository hides where data comes from; the view model manages refresh state and messages.
- `NWPathMonitor` for connectivity hints; always attempt requests anyway.
- A larger `URLCache` keeps images around.
:::

:::exercise
1. Only refresh automatically when the cache is older than 10 minutes (pull-to-refresh always refreshes). Hint: compare `lastUpdated` with `Date.now.addingTimeInterval(-600)`.
2. Add a `NetworkMonitor` to the environment and show an "Offline" banner at the top of the list when disconnected. Refresh automatically when the connection comes back (`.onChange(of: monitor.isConnected)`).
:::

## Check your understanding

```quiz
Q: In an offline-first screen, what should the user see first when opening the app?
- [ ] A spinner until the network responds
- [x] The cached data, immediately, while fresh data loads
- [ ] An empty list
- [ ] An error if offline
> Showing cached data instantly is the core of offline-first.

Q: A refresh fails but cached data is on screen. What should happen?
- [ ] Clear the list and show an error screen
- [x] Keep the cached data and show a small, non-blocking message
- [ ] Crash
- [ ] Retry in a tight loop forever
> Never throw away usable data because of a failed refresh.

Q: Why is DiskCache an actor?
- [ ] Actors are faster than structs
- [x] So concurrent reads and writes from different tasks can't clash
- [ ] Files can only be written from actors
- [ ] To run on the main thread
> Actors serialise access to shared mutable resources.

Q: Should you skip a request because NWPathMonitor says you're offline?
- [ ] Yes, always
- [x] No. Attempt the request and handle failure; use the monitor for UI hints and retries
- [ ] Only on cellular
- [ ] Only for POST requests
> Reachability can be wrong; the request itself is the real test.
```
