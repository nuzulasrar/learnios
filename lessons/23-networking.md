---
id: networking
title: Networking with URLSession & Codable
part: 5
minutes: 40
summary: Fetch real data from the internet. HTTP and JSON basics, decoding with Codable, URLSession with async/await, handling errors and status codes, and showing it all in SwiftUI.
---

:::goals
- HTTP, REST and JSON in a nutshell
- Decoding JSON into Swift types with `Codable`
- Custom keys, nested JSON, dates and optional fields
- `URLSession` with `async`/`await`
- Checking status codes and handling errors properly
- Sending data with POST
:::

## HTTP & JSON in 60 seconds

Apps talk to servers over **HTTP**. A REST API exposes data at URLs called *endpoints*. You send a **request** (`GET` to read, `POST` to create, `PUT`/`PATCH` to update, `DELETE` to remove) and get a **response** with a **status code** (`200` OK, `404` not found, `500` server error) and usually a **JSON** body.

We'll use **[DummyJSON](https://dummyjson.com)**, a free test API that needs no key. Open this in your browser:

```text
https://dummyjson.com/quotes/random
```

```json
{
  "id": 378,
  "quote": "If you tell the truth, you don't have to remember anything.",
  "author": "Mark Twain"
}
```

:::note HTTPS only
iOS's **App Transport Security** blocks plain `http://` connections by default. Always use `https://`. (Exceptions can be configured in Info.plist, but avoid them.) Unlike some platforms, iOS apps need no special permission to use the internet.
:::

## Decoding JSON with `Codable`

Define a struct whose properties match the JSON keys, and conform to **`Decodable`** (or **`Codable`**, which is Decodable + Encodable). Swift generates all the parsing code:

```swift title="Quote.swift"
import Foundation

struct Quote: Codable, Identifiable, Hashable {
    let id: Int
    let quote: String
    let author: String
}
```

```swift runnable
import Foundation

struct Quote: Codable, Identifiable {
    let id: Int
    let quote: String
    let author: String
}

let json = """
{ "id": 378, "quote": "If you tell the truth, you don't have to remember anything.", "author": "Mark Twain" }
""".data(using: .utf8)!

do {
    let quote = try JSONDecoder().decode(Quote.self, from: json)
    print("“\(quote.quote)” — \(quote.author)")
} catch {
    print("Decoding failed: \(error)")
}
```

### Nested JSON, renamed keys, optional fields

Real APIs rarely match your ideal model. Codable handles it:

```swift runnable
import Foundation

struct QuotePage: Decodable {
    let quotes: [QuoteDTO]          // nested array
    let total: Int
}

struct QuoteDTO: Decodable {
    let id: Int
    let text: String                // we prefer "text" over the API's "quote"
    let author: String
    let tags: [String]?             // optional: may be missing from the JSON

    enum CodingKeys: String, CodingKey {
        case id, author, tags
        case text = "quote"         // map JSON "quote" → Swift `text`
    }
}

let json = """
{
  "quotes": [
    { "id": 1, "quote": "Your heart is the size of an ocean.", "author": "Rumi" },
    { "id": 2, "quote": "Stay hungry, stay foolish.", "author": "Stewart Brand", "tags": ["life"] }
  ],
  "total": 1454,
  "skip": 0,
  "limit": 2
}
""".data(using: .utf8)!

let page = try JSONDecoder().decode(QuotePage.self, from: json)
for q in page.quotes {
    print(q.id, q.text, "—", q.author, q.tags ?? [])
}
print("Total quotes on the server:", page.total)
```

- **Extra JSON keys are ignored** (`skip` and `limit` above), so APIs can add fields without breaking your app.
- A **missing key** for a non-optional property throws an error, so make properties optional when the API might omit them.
- For `snake_case` APIs, set `decoder.keyDecodingStrategy = .convertFromSnakeCase` so `image_url` maps to `imageUrl` automatically.
- For dates, set `decoder.dateDecodingStrategy = .iso8601`.

## Fetching with `URLSession`

```swift title="QuoteService.swift"
import Foundation

struct QuoteDTO: Decodable {
    let id: Int
    let quote: String
    let author: String
}

enum APIError: LocalizedError {
    case badStatus(Int)
    case decoding

    var errorDescription: String? {
        switch self {
        case .badStatus(let code): "The server responded with an error (\(code))."
        case .decoding: "The data from the server was in an unexpected format."
        }
    }
}

struct QuoteService {
    private let baseURL = URL(string: "https://dummyjson.com")!
    private let session: URLSession = .shared

    func randomQuote() async throws -> QuoteDTO {
        let url = baseURL.appending(path: "quotes/random")
        let (data, response) = try await session.data(from: url)   // suspends, never blocks the UI

        guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
            throw APIError.badStatus((response as? HTTPURLResponse)?.statusCode ?? -1)
        }
        do {
            return try JSONDecoder().decode(QuoteDTO.self, from: data)
        } catch {
            throw APIError.decoding
        }
    }

    func quotes(limit: Int = 20, skip: Int = 0) async throws -> [QuoteDTO] {
        var components = URLComponents(url: baseURL.appending(path: "quotes"), resolvingAgainstBaseURL: false)!
        components.queryItems = [                       // builds ?limit=20&skip=0 safely
            URLQueryItem(name: "limit", value: String(limit)),
            URLQueryItem(name: "skip", value: String(skip))
        ]
        let (data, _) = try await session.data(from: components.url!)
        struct Page: Decodable { let quotes: [QuoteDTO] }
        return try JSONDecoder().decode(Page.self, from: data).quotes
    }
}
```

Key points:

- `try await session.data(from:)` returns the response body (`Data`) and metadata (`URLResponse`).
- **Always check the status code.** URLSession doesn't throw for `404` or `500`; it only throws for transport problems (no internet, timeout, cancelled).
- Build URLs with `appending(path:)` and `URLComponents` instead of gluing strings together. That handles escaping for you.

## Showing it in SwiftUI

Combine the service with the view model pattern from Lesson 21:

```swift title="RandomQuoteView.swift"
import SwiftUI

@MainActor
@Observable
final class RandomQuoteViewModel {
    enum State { case loading, loaded(QuoteDTO), failed(String) }

    private(set) var state: State = .loading
    private let service = QuoteService()

    func load() async {
        state = .loading
        do {
            state = .loaded(try await service.randomQuote())
        } catch is CancellationError {
            // the view went away, nothing to show
        } catch {
            state = .failed(error.localizedDescription)
        }
    }
}

struct RandomQuoteView: View {
    @State private var viewModel = RandomQuoteViewModel()

    var body: some View {
        VStack(spacing: 20) {
            switch viewModel.state {
            case .loading:
                ProgressView()
            case .loaded(let quote):
                Text("“\(quote.quote)”").font(.title2).multilineTextAlignment(.center)
                Text("— \(quote.author)").foregroundStyle(.secondary)
            case .failed(let message):
                ContentUnavailableView("Couldn't load a quote", systemImage: "wifi.slash",
                                       description: Text(message))
            }
            Button("Another quote") { Task { await viewModel.load() } }
                .buttonStyle(.borderedProminent)
        }
        .padding()
        .task { await viewModel.load() }
    }
}

#Preview { RandomQuoteView() }
```

Run it (the preview works too, and makes a real network request). Turn on airplane mode in the Simulator's host Mac, or use a bad URL, to see the error state.

:::tip Friendly error messages
`URLError` has useful codes. `.notConnectedToInternet`, `.timedOut` and `.cannotFindHost` can become messages like "You're offline". Never show raw error dumps to users.
:::

## Sending data: POST with a JSON body

```swift title="PostExample.swift"
import Foundation

struct NewPost: Encodable {
    let title: String
    let userId: Int
}

struct CreatedPost: Decodable {
    let id: Int
    let title: String
}

func createPost(title: String) async throws -> CreatedPost {
    var request = URLRequest(url: URL(string: "https://dummyjson.com/posts/add")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(NewPost(title: title, userId: 5))

    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 201 else {
        throw URLError(.badServerResponse)
    }
    return try JSONDecoder().decode(CreatedPost.self, from: data)
}
```

For APIs that need authentication, add a header: `request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")`. Store tokens in the Keychain (Lesson 25), never in code or UserDefaults.

## Debugging network calls

- `print(String(decoding: data, as: UTF8.self))` shows the raw JSON while you debug.
- When decoding fails, `print(error)` on a `DecodingError` tells you exactly which key or type didn't match.
- Test endpoints in a browser or with `curl` first to see the real JSON shape.

## Recap

:::recap
- `Codable` structs mirror JSON; use `CodingKeys` to rename, and optionals for keys that may be missing.
- `try await URLSession.shared.data(from:)`: check the **status code** yourself.
- Build URLs with `appending(path:)` and `URLComponents`.
- Map errors to friendly messages; ignore `CancellationError`.
- `URLRequest` + `JSONEncoder` for POST.
:::

## Practice

:::exercise
1. Build a **Quotes list** screen: load 30 quotes with `quotes(limit:skip:)` and show them in a `List`, with pull-to-refresh loading a random page (`skip: Int.random(in: 0...1400)`).
2. Show `ContentUnavailableView` with a **Try again** button on failure.
3. Add `.searchable` to filter the loaded quotes by author, locally.
:::

:::solution Show solution
```swift
import SwiftUI

@MainActor
@Observable
final class QuoteListViewModel {
    private(set) var quotes: [QuoteDTO] = []
    private(set) var errorMessage: String?
    private(set) var isLoading = false
    var searchText = ""
    private let service = QuoteService()

    var filtered: [QuoteDTO] {
        searchText.isEmpty ? quotes : quotes.filter { $0.author.localizedCaseInsensitiveContains(searchText) }
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        do {
            quotes = try await service.quotes(limit: 30, skip: Int.random(in: 0...1400))
            errorMessage = nil
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

struct QuoteListView: View {
    @State private var viewModel = QuoteListViewModel()

    var body: some View {
        NavigationStack {
            List(viewModel.filtered, id: \.id) { q in
                VStack(alignment: .leading, spacing: 4) {
                    Text("“\(q.quote)”")
                    Text(q.author).font(.caption).foregroundStyle(.secondary)
                }
            }
            .overlay {
                if let message = viewModel.errorMessage, viewModel.quotes.isEmpty {
                    ContentUnavailableView {
                        Label("Offline", systemImage: "wifi.slash")
                    } description: {
                        Text(message)
                    } actions: {
                        Button("Try again") { Task { await viewModel.load() } }
                    }
                } else if viewModel.isLoading && viewModel.quotes.isEmpty {
                    ProgressView()
                }
            }
            .searchable(text: $viewModel.searchText, prompt: "Filter by author")
            .refreshable { await viewModel.load() }
            .task { await viewModel.load() }
            .navigationTitle("Quotes")
        }
    }
}

#Preview { QuoteListView() }
```
:::

## Check your understanding

```quiz
Q: What happens when the JSON contains keys your Decodable struct doesn't declare?
- [ ] Decoding fails
- [x] They're ignored
- [ ] They're added as properties
- [ ] The app crashes
> Codable ignores unknown keys, but missing required keys do throw.

Q: Does `URLSession.data(from:)` throw when the server returns 404?
- [ ] Yes, always
- [x] No. You must check the HTTPURLResponse status code yourself
- [ ] Only on iOS 27
- [ ] Only for POST requests
> Transport errors throw; HTTP error statuses don't.

Q: The API sends `"quote"` but you want the Swift property to be called `text`. What do you use?
- [ ] Rename it in the JSON
- [x] A CodingKeys enum with `case text = "quote"`
- [ ] keyDecodingStrategy = .convertFromSnakeCase
- [ ] A computed property only
> CodingKeys maps JSON keys to property names.

Q: Why use `URLComponents` with `queryItems` instead of string concatenation?
- [ ] It's required for HTTPS
- [x] It correctly escapes special characters in query values
- [ ] It makes requests faster
- [ ] String URLs are deprecated
> Building URLs by hand breaks on spaces, ampersands and non-ASCII text.
```
