<div align="center">
  <img width="128" height="128" src="/resources/icon.png" alt="Chronicle Icon">
  <h1><b>Chronicle</b></h1>
  <p>
    Advanced logging wrapper for Apple's unified logging with async, structured metadata, runtime details and more.
  </p>
</div>

<p align="center">
  <a href="https://swift.org"><img src="https://img.shields.io/badge/Swift-6.0+-F05138?logo=swift&logoColor=white" alt="Swift 6.0+"></a>
  <a href="https://developer.apple.com"><img src="https://img.shields.io/badge/iOS-13+-000000?logo=apple" alt="iOS 13+"></a>
  <a href="https://developer.apple.com"><img src="https://img.shields.io/badge/macOS-10.15+-000000?logo=apple" alt="macOS 10.15+"></a>
  <a href="https://developer.apple.com"><img src="https://img.shields.io/badge/tvOS-13+-000000?logo=apple" alt="tvOS 13+"></a>
  <a href="https://developer.apple.com"><img src="https://img.shields.io/badge/watchOS-6+-000000?logo=apple" alt="watchOS 6+"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License: MIT"></a>
</p>


## Overview

- Adaptive backend: Apple's modern `os.Logger` on supported OS versions, seamless `os_log` fallback otherwise
- A global, shared `Logger` singleton plus multiple keyed shared instances for packages/modules
- Extensible, enum-style `Tag`s and JSON-like structured metadata via `LogMetadataValue`
- Runtime filtering by `LogLevel`, async logging helper, and zero-overhead disabled logs
- Optional in-app console with SwiftUI view, environment modifier, and composable store
- Signpost convenience APIs for performance tracing
- Optional SwiftLog integration (`LoggingSystem.bootstrapChronicle`)


## Installation

```swift
dependencies: [
    .package(url: "https://github.com/aeastr/Chronicle.git", from: "3.0.0")
]
```

```swift
import Chronicle
```

| Target | Description |
|--------|-------------|
| `Chronicle` | Core logging module |
| `ChronicleConsole` | Optional SwiftUI in-app console |
| `ChronicleSwiftLogBridge` | Optional swift-log integration for server-side Swift |


## Usage

### Basic Logging

```swift
import Chronicle

// Define your tags
extension Tag {
    static let cache   = Tag("Cache")
    static let network = Tag("Network")
    static let ui      = Tag("UI")
}

// Configure at app launch
Logger.shared.subsystem = Bundle.main.bundleIdentifier ?? "com.myapp"

#if DEBUG
Logger.shared.setAllowedLevels(Set(LogLevel.allCases))
#else
Logger.shared.setAllowedLevels([.error, .fault])
#endif

// Emit logs
Logger.shared.log("Loaded from disk", level: .info, tags: [.cache])

Logger.shared.log(
    "Request failed",
    level: .error,
    tags: [.network],
    metadata: [
        "url": request.url,
        "userID": user.id,
        "retry": false
    ]
)
```

### Multiple Logger Instances

Create separate logger instances for packages, modules, or features with independent configuration:

```swift
// Default global logger
Logger.shared.log("App log", level: .info)

// Named loggers with independent filtering
let packageLogger = Logger.shared(for: "com.example.package")
packageLogger.setAllowedLevels([.debug, .info])
packageLogger.log("Log from package", level: .debug)

let networkLogger = Logger.shared(for: "com.example.network")
networkLogger.setAllowedLevels([.error, .fault])
networkLogger.log("Network error", level: .error)
```

### Async Logging

```swift
await Logger.shared.logAsync(level: .debug, tags: [.network]) {
    try await DetailedRequestDebugger.makeSummary(for: transaction)
}
```

### Structured Metadata

Metadata supports strings, integers, doubles, booleans, arrays, dictionaries, and null values:

```swift
Logger.shared.log(
    "Cache miss",
    level: .notice,
    tags: [.cache],
    metadata: [
        "key": "user_42",
        "reason": "Expired",
        "context": [
            "policy": "LRU",
            "lastHit": 1_714_882_233,
            "counts": [1, 3, 5]
        ]
    ]
)
```

Output:

```
[Cache] Cache miss | {"key":"user_42","reason":"Expired","context":{"policy":"LRU","lastHit":1714882233,"counts":[1,3,5]}}
```

### Signposts

Trace performance-critical paths:

```swift
let id = Logger.shared.beginSignpost("Image Decode", tags: [.ui])
defer { Logger.shared.endSignpost("Image Decode", id: id) }
```


## Customization

### Output Options

```swift
var options = Logger.shared.currentOutputOptions
options.showMetadata = false
options.metadataFormat = .json
options.metadataKeyPolicy = .include(["scenePhase", "debugOverlays"])
Logger.shared.setOutputOptions(options)

Logger.shared.updateOutputOptions { options in
    options.showMetadata = true
    options.showSource = false
}
```

### Event Sinks

Register additional sinks to observe logs in real time:

```swift
let token = Logger.shared.addEventSink { entry in
    print("[\(entry.level)] \(entry.message)")
}

// Later, remove when no longer needed
Logger.shared.removeEventSink(token)
```

### In-App Console

Enable the SwiftUI console for QA or support builds:

```swift
import ChronicleConsole

// Enable the console store
let consoleStore = Logger.shared.enableConsole(maxEntries: 1_000)

// Wire into SwiftUI
struct RootView: View {
    @State private var showConsole = false

    var body: some View {
        Content()
            .logConsole(enabled: true)
            .toolbar {
                Button("Console") { showConsole = true }
            }
            .sheet(isPresented: $showConsole) {
                if #available(iOS 16.0, macOS 13.0, tvOS 16.0, *) {
                    LogConsolePanel()
                }
            }
    }
}
```

Console capabilities: pause/resume live updates, toggle metadata/timestamp display, copy/share entries, clear buffered logs.

### SwiftLog Integration

For server-side Swift or `swift-log` codebases:

```swift
import Logging
import ChronicleSwiftLogBridge

LoggingSystem.bootstrapChronicle(defaultLogLevel: .info)

let logger = Logger(label: "com.myapp.feature")
logger.notice("Task queued", metadata: ["id": .string(task.id)])
```


## How It Works

Chronicle wraps Apple's Unified Logging system (`os.Logger` / `os_log`) with a Swift-friendly API. Key types:

| Type | Purpose |
|------|---------|
| `Logger` | Main entry point with shared singleton and registry for named instances |
| `Tag` | Type-safe string wrapper for categorizing logs |
| `LogLevel` | Maps to `OSLogType`: debug, info, notice, warning, error, fault |
| `LogMetadataValue` | Recursive enum for structured JSON-like metadata |
| `LogOutputOptions` | Controls metadata rendering and key filtering |

All logger APIs are concurrency-safe for Swift 6. Use `logAsync` when you want to await expensive message builders without blocking callers.


## Contributing

Contributions welcome. Please feel free to submit a Pull Request. See the [Contributing Guide](CONTRIBUTING.md) for details.


## License

MIT. See [LICENSE](LICENSE.md) for details.
