---
description: Architectural reference for ServerFileBrowser
---

# ServerFileBrowser

**Package:** com.hypixel.hytale.server.core.ui.browser
**Type:** Stateful Component

## Definition
```java
// Signature
public class ServerFileBrowser {
```

## Architecture & Concepts
The ServerFileBrowser is a server-side controller that provides the backend logic for a file browsing user interface. It operates as a state machine, managing the user's current navigation state, including the root directory, current path, search queries, and selected items.

This class is a key component in Hytale's server-driven UI architecture. It does not perform any rendering itself. Instead, it translates its internal state into a series of commands for the client UI framework using a `UICommandBuilder` and `UIEventBuilder`. It receives user input through a structured `FileBrowserEventData` object, processes the event, updates its internal state, and prepares for the next UI build cycle.

Its behavior is heavily customized through a `FileBrowserConfig` object provided during construction, allowing developers to define permissible file roots, allowed extensions, and UI element bindings without modifying the browser's core logic. This makes ServerFileBrowser a reusable, data-driven component for any feature requiring file system interaction, such as world selection, asset importing, or configuration management.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor: `new ServerFileBrowser(config)`. It is not managed by a dependency injection container or a central service registry. Ownership is typically held by a higher-level UI manager or game state controller responsible for the screen that displays the file browser.
- **Scope:** The object's lifetime is transient and bound to a specific UI session. It is created when a player opens a file browser interface and should be discarded when the interface is closed.
- **Destruction:** The object is cleaned up by the Java garbage collector once all references to it are released. It holds no native resources and does not require an explicit destruction method.

## Internal State & Concurrency
- **State:** ServerFileBrowser is highly stateful and mutable. It maintains the current `root` path, `currentDir` relative to the root, the active `searchQuery`, and a set of `selectedItems`. File listings are not cached; they are regenerated from the file system on every call to `buildFileList`.
- **Thread Safety:** This class is **not thread-safe**. All methods assume they are invoked from a single, consistent thread, such as the main server tick loop or a dedicated player-session thread. Concurrent access from multiple threads will lead to race conditions, inconsistent UI state, and potential crashes. All interactions with an instance of this class must be synchronized externally.

## API Surface
The public API provides two primary functions: building UI commands from the current state and handling events to modify that state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| buildUI(cmdBuilder, eventBuilder) | void | O(N log N) | Populates UI builders with commands to render the browser. Complexity is dominated by file system listing and sorting. |
| handleEvent(eventData) | boolean | O(1) | Processes user input, such as navigation or search. Returns true if the event was consumed and resulted in a state change. |
| resolveSecure(relativePath) | Path | O(1) | Safely resolves a relative path against the root, preventing path traversal attacks. Returns null if the path is invalid. |
| navigateTo(relativePath) | void | O(1) | Changes the current directory. Includes security checks to prevent navigation outside the root. |
| navigateUp() | void | O(1) | Navigates to the parent of the current directory. |

## Integration Patterns

### Standard Usage
The typical lifecycle involves creating the browser, then repeatedly handling events and building the UI in a loop until the user completes their interaction.

```java
// 1. Configure and create the browser instance
FileBrowserConfig config = new FileBrowserConfig(...);
ServerFileBrowser browser = new ServerFileBrowser(config);

// 2. In an event handler, process incoming data
// This typically happens when a packet is received from the client.
FileBrowserEventData eventData = FileBrowserEventData.fromPacket(packet);
boolean stateChanged = browser.handleEvent(eventData);

// 3. In the UI update phase, generate commands for the client
if (stateChanged) {
    UICommandBuilder commandBuilder = new UICommandBuilder();
    UIEventBuilder eventBuilder = new UIEventBuilder();
    browser.buildUI(commandBuilder, eventBuilder);
    
    // Send the generated commands to the client
    player.sendUICommands(commandBuilder.build());
    player.sendUIEvents(eventBuilder.build());
}
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Do not call methods on a ServerFileBrowser instance from multiple threads. All interactions must be serialized onto a single thread.
- **Unsafe Path Resolution:** Do not manually construct file paths from user input. Always use `resolveSecure` or `resolveFromCurrent` to validate that a selected file path is within the configured root directory. Failure to do so creates a severe security vulnerability.
- **State Leakage:** Do not hold references to a ServerFileBrowser instance beyond the lifetime of the UI screen it manages. This will prevent garbage collection and lead to a memory leak.

## Data Pipeline
ServerFileBrowser acts as a bridge between client-side UI events and server-side file system logic. Its role in the data flow is cyclical.

> **Event Handling Flow (Input):**
> Client UI Interaction -> Network Packet -> Server Packet Handler -> `FileBrowserEventData` -> **ServerFileBrowser.handleEvent()** -> Internal State Mutation

> **UI Building Flow (Output):**
> Server Update Tick -> **ServerFileBrowser.buildUI()** -> `UICommandBuilder` & `UIEventBuilder` -> UI Commands -> Network Packet -> Client UI Render Engine

