# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Mac Catalyst app that exposes Apple HomeKit as an MCP server over stdio (JSON-RPC 2.0), using the official Swift MCP SDK (`modelcontextprotocol/swift-sdk`, pinned in `Package.resolved`). MCP clients launch the app binary directly: `HomeKitMCP.app/Contents/MacOS/HomeKitMCP --headless`.

## Build and test

There is no CLI build script; everything goes through the Xcode project. HomeKit requires a signed build with entitlements, so builds need `Local.xcconfig` (gitignored; copy `Local.xcconfig.template`). It sets `DEVELOPMENT_TEAM` and `BUNDLE_ID_PREFIX`; all three targets' bundle IDs are `$(BUNDLE_ID_PREFIX).<Target>`.

Signing gotcha: `<prefix>.HomeKitMCP` must exist as an explicit App ID with the HomeKit capability in the developer portal. If it does not, automatic signing silently uses the team wildcard profile and fails with "Entitlement com.apple.developer.homekit not found". `-allowProvisioningUpdates` does not register the App ID for you.

```bash
# Build (Mac Catalyst destination is required)
xcodebuild -project HomeKitMCP.xcodeproj -scheme HomeKitMCP \
  -destination 'platform=macOS,variant=Mac Catalyst' -allowProvisioningUpdates build

# Run all tests (Swift Testing, hosted in the app, so tests also need signing)
xcodebuild -project HomeKitMCP.xcodeproj -scheme HomeKitMCP \
  -destination 'platform=macOS,variant=Mac Catalyst' -allowProvisioningUpdates test

# Run one suite or one test
xcodebuild ... test -only-testing:HomeKitMCPTests/MCPServerManagerTests
xcodebuild ... test -only-testing:HomeKitMCPTests/MCPServerManagerTests/listHomesRoutesToHomeKitManager
```

Tests use Swift Testing (`import Testing`, `@Test`, `#expect`), not XCTest.

CI (`.github/workflows/test.yml`) runs the same `xcodebuild test` on a hosted `macos-26` runner. It uses ad-hoc signing with no team and no entitlements, which works because the tests use `MockHomeKitManager` and never touch HomeKit. The job name `test` is the sanity check that `dependabot-auto-merge.yml` names, so keep the two in sync.

## Architecture

Three Xcode targets, all using file-system-synchronized groups (new files in a target's folder are added to that target automatically):

- **HomeKitMCP** (Catalyst app, `SDKROOT = iphoneos`, `SUPPORTS_MACCATALYST`)
- **StatusBarPlugin** (macOS AppKit `.bundle`, embedded in the app's `PlugIns/`)
- **HomeKitMCPTests** (unit tests, `TEST_HOST` is the app)

### Request flow

`MCPServerManager` (`HomeKitMCP/MCP/`) owns the MCP `Server` and `StdioTransport`. `ListTools` returns `ToolDefinitions.all`; `CallTool` goes to `handleToolCall`, a single `switch` on tool name. That switch validates arguments, calls `HomeKitProviding`, and serializes the returned `[String: Any]` with `JSONSerialization`. With swift-sdk 0.12+, `Tool.Content.text` has three associated values; pattern-match it as `case .text(let text, _, _)`. Versions below 0.12.1 do not compile with the Xcode 27 Swift compiler. `convertValue` turns MCP `Value` into Foundation types. It also re-parses string values that hold JSON objects or arrays, because models sometimes stringify nested arguments.

`HomeKitManager` (`HomeKitMCP/HomeKit/`) is the only code that touches the HomeKit framework (`HMHomeManager`). It is `@MainActor`. `start()` suspends until the `homeManagerDidUpdateHomes` delegate callback fires. It maps HomeKit service types to device-type strings (`deviceType(for:)`) and characteristics to JSON keys (`characteristicKey(for:)`). Errors are `HomeKitError: LocalizedError`; their descriptions are what MCP clients see.

### Adding or changing a tool

A tool touches four places, which must stay in sync:

1. `ToolDefinitions.swift`: the `Tool` schema, and add it to `ToolDefinitions.all`.
2. `MCPServerManager.handleToolCall`: the `case` with argument validation.
3. `HomeKitProviding` protocol and `HomeKitManager` implementation.
4. `HomeKitMCPTests/MockHomeKitManager.swift`, plus tests. `ToolDefinitionsTests.allToolsCount` asserts the tool count.

Update the tool and device-type tables in `README.md` too.

### Lifecycle and process exit

- `HomeKitMCPApp` creates `MCPServerManager(startServer: false)` and starts the server from a view `.task`. That way the test host process never starts a stdio server. Tests construct `MCPServerManager(homeKitManager: mock, startServer: false)` and call `handleToolCall` directly.
- The process calls `exit()` when stdin closes, on SIGINT/SIGTERM (handled through `DispatchSource`), and on `willTerminate`. Code after `server.waitUntilCompleted()` does not return.
- **stdout is the JSON-RPC channel.** All logging goes to stderr through `Log` (`Utilities/Logger.swift`). Never `print()` to stdout.

### Menu bar plugin (AppKit from Catalyst)

Catalyst cannot use `NSStatusItem` directly. Without `--headless`, `StatusBarLoader` loads `PlugIns/StatusBarPlugin.bundle` at runtime. It instantiates the bundle's `NSPrincipalClass` (`StatusBarController`) and casts it to the `@objc StatusBarBridgeProtocol`. `StatusBarBridge.swift` lives in the app folder but is also compiled into StatusBarPlugin through a membership exception in the project file. Keep the protocol `@objc` and identical on both sides. With `--headless`, the app hides its windows directly and no plugin loads.
