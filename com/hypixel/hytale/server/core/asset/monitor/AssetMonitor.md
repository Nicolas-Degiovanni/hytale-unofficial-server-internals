---
description: Architectural reference for AssetMonitor
---

# AssetMonitor

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Singleton-like Service

## Definition
```java
// Signature
public class AssetMonitor {
```

## Architecture & Concepts

The AssetMonitor is a high-performance, asynchronous service responsible for monitoring file system directories for changes. Its primary role within the server architecture is to enable live reloading of game assets during development, providing a rapid iteration workflow for content creators and engineers.

It operates as a centralized system that decouples low-level file system notifications from high-level asset management logic. The core design is built upon a multi-stage, event-driven pipeline that processes raw file events through debouncing and batching phases before notifying registered consumers.

This system is foundational for any part of the engine that needs to react to external file modifications, such as reloading models, textures, configuration files, or scripts without requiring a full server restart. It uses a dedicated background thread for watching and a scheduled executor for task processing to ensure that file I/O and monitoring logic do not block the main server loop.

### Lifecycle & Ownership
-   **Creation:** An instance is created via its public constructor, `new AssetMonitor()`. This is typically performed once during the server's bootstrap sequence. The constructor immediately allocates critical resources, including starting the `PathWatcherThread`, making the object active upon instantiation.
-   **Scope:** The AssetMonitor is a long-lived object, designed to persist for the entire lifetime of the server process.
-   **Destruction:** The `shutdown()` method **must** be called explicitly during the server shutdown sequence. Failure to do so will result in resource leaks and prevent the JVM from terminating, as the background threads are not daemon threads by default (though the executor uses a daemon thread factory).

## Internal State & Concurrency
-   **State:** The AssetMonitor is a highly stateful component. Its internal state consists of several concurrent maps that track monitored directories, registered handlers, and pending change-processing tasks. This state is mutable and constantly updated by both external API calls and internal event processing threads.
-   **Thread Safety:** This class is designed to be thread-safe for its public API. Registration methods like `monitorDirectoryFiles` can be safely called from any thread. It orchestrates work across three distinct thread contexts:
    1.  **Caller Thread:** The thread invoking `monitorDirectoryFiles` or `removeMonitorDirectoryFiles`.
    2.  **PathWatcherThread:** A dedicated internal thread that blocks on and receives raw file system events from the operating system.
    3.  **AssetMonitor Thread:** A single-threaded `ScheduledExecutorService` used to execute delayed tasks for debouncing and batching.

    State management is primarily handled through the use of `ConcurrentHashMap`.

    **Warning:** Handlers provided to this service will be executed on the shared, single-threaded `AssetMonitor Thread`. Any handler that performs a long-running or blocking operation will delay or starve all other asset change notifications system-wide. Handlers must be designed to be lightweight and execute quickly.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| monitorDirectoryFiles(Path path, AssetMonitorHandler handler) | void | O(1) avg | Registers a directory for monitoring. Associates a handler to be invoked on changes. Throws IllegalArgumentException if the path is not a directory. |
| removeMonitorDirectoryFiles(Path path, Object key) | void | O(N) | Removes a handler associated with the given key from a monitored directory. N is the number of handlers for the path. |
| shutdown() | void | O(1) | Terminates the background watcher thread and shuts down the executor service. This is a critical cleanup step. |

## Integration Patterns

### Standard Usage
The AssetMonitor should be treated as a central service. Other systems that manage file-based assets retrieve the AssetMonitor instance and register a path and a handler. The handler contains the logic to reload or update the relevant asset.

```java
// Example: A ModelManager service setting up live-reloading
// Assume 'assetMonitor' is a singleton instance obtained at startup.

Path modelDirectory = Paths.get("server/assets/models");
AssetMonitorHandler modelReloadHandler = new ModelReloadHandler(); // Implements AssetMonitorHandler

// The ModelManager registers its interest in the models directory.
// The AssetMonitor will now invoke the handler when files change.
assetMonitor.monitorDirectoryFiles(modelDirectory, modelReloadHandler);
```

### Anti-Patterns (Do NOT do this)
-   **Multiple Instances:** Do not create more than one AssetMonitor per application. Each instance spawns its own resource-intensive background threads. The system is designed to be a single, centralized service.
-   **Forgetting Shutdown:** Never allow an AssetMonitor instance to fall out of scope without calling `shutdown()`. This is a guaranteed resource leak that will prevent the application from exiting cleanly.
-   **Blocking Handlers:** Do not perform file I/O, network requests, or any other blocking operations directly within an `AssetMonitorHandler`. This will block the single executor thread and halt all other asset notifications. Delegate heavy work to a separate thread pool.

## Data Pipeline
The flow of data from a file system event to a handler invocation is a multi-step, asynchronous process designed to reduce noise and improve efficiency.

> Flow:
> OS File System Event -> `PathWatcherThread` -> **AssetMonitor::onChange** (Debouncing via `FileChangeTask`) -> `ScheduledExecutorService` -> **AssetMonitor::onDelayedChange** (Dispatching) -> `DirectoryHandlerChangeTask` (Batching) -> `AssetMonitorHandler::onChange` (Consumer Logic)

