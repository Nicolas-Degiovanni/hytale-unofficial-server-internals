---
description: Architectural reference for PathWatcherThread
---

# PathWatcherThread

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Transient Service

## Definition
```java
// Signature
public class PathWatcherThread implements Runnable {
```

## Architecture & Concepts

The PathWatcherThread is a low-level, high-performance service responsible for monitoring file system directories for changes. It serves as a foundational component for systems that require real-time updates when underlying files are created, modified, or deleted, such as the server's asset hot-reloading system.

Architecturally, this class acts as an abstraction layer over Java's NIO WatchService API. Its primary design goals are to:
1.  **Isolate Blocking I/O:** All file system monitoring is performed on a dedicated background daemon thread, ensuring the main server thread is never blocked waiting for file events.
2.  **Decouple Event Detection from Handling:** It uses a callback-based design, accepting a BiConsumer in its constructor. The PathWatcherThread is only responsible for *detecting* an event; the provided consumer is responsible for implementing the *logic* to handle it.
3.  **Abstract Platform Differences:** It provides a unified interface for watching directories, automatically handling significant underlying OS differences. On Windows, it leverages the efficient `FILE_TREE` modifier to watch an entire directory hierarchy with a single native handle. On other platforms like Linux or macOS, it transparently falls back to recursively walking the directory and registering a separate watcher for each subdirectory.

This component is critical for developer workflows, enabling live changes to assets without requiring a full server restart.

### Lifecycle & Ownership

-   **Creation:** An instance is created manually by a higher-level system, such as an AssetService or a development-mode manager. The creator must provide a non-null BiConsumer which will be invoked with the path and type of file system event.
-   **Scope:** The object and its associated background thread persist as long as file monitoring is required. It is designed for long-running sessions. The internal thread is configured as a daemon, which means it will not prevent the JVM from shutting down.
-   **Destruction:** The lifecycle must be managed explicitly by the owner. The `shutdown` method must be called to guarantee a graceful termination of the background thread and the release of all native file system handles held by the WatchService. Failure to call `shutdown` will result in resource leaks.

## Internal State & Concurrency

-   **State:** This class is stateful. Its primary state is the `registered` map, a ConcurrentHashMap that tracks which directory paths are currently being monitored and their corresponding WatchKey. This state is mutable, primarily modified when new paths are added via the `addPath` method.
-   **Thread Safety:** The class is designed to be thread-safe. Public methods such as `addPath` and `shutdown` can be safely called from any thread. The core event processing loop runs exclusively on its own internal "PathWatcher" thread.

    **WARNING:** The BiConsumer callback provided during construction is executed **on the internal PathWatcher thread**. Any logic within this consumer must be thread-safe. Modifying shared game state or other non-thread-safe collections from this callback will introduce severe race conditions. The recommended pattern is to use the consumer to queue a task for execution on the main server thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PathWatcherThread(consumer) | constructor | O(1) | Creates the watcher service and a new daemon thread. Does not start monitoring. Throws IOException on failure. |
| start() | void | O(1) | Starts the background thread, which begins waiting for file system events. |
| shutdown() | void | O(1) | Initiates a graceful shutdown. Interrupts the thread and waits up to 1 second for it to terminate. |
| addPath(path) | void | O(N) | Registers a directory path for monitoring. Complexity is O(1) on Windows but O(N) on other systems, where N is the number of subdirectories, due to recursive registration. |

## Integration Patterns

### Standard Usage

The PathWatcherThread must be explicitly managed. The recommended pattern is to create, start, and subsequently shut it down within a try-finally block or an equivalent lifecycle management system to prevent resource leaks.

```java
// The consumer logic should be thread-safe or delegate to the main thread
BiConsumer<Path, EventKind> myAssetHandler = (path, eventKind) -> {
    System.out.println("File event: " + eventKind + " on path: " + path);
    // In a real application, you would queue this work for the main game thread
};

PathWatcherThread watcher = null;
try {
    watcher = new PathWatcherThread(myAssetHandler);
    watcher.addPath(Paths.get("./assets/models"));
    watcher.start();

    // ... server runs for its duration ...

} catch (IOException e) {
    // Handle initialization failure
} finally {
    if (watcher != null) {
        watcher.shutdown();
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Forgetting Shutdown:** Failing to call `shutdown()` will leak native file handles managed by the WatchService, which can lead to resource exhaustion.
-   **Blocking in the Consumer:** Performing long-running or blocking operations inside the BiConsumer callback will stall the PathWatcher thread. This will prevent it from processing subsequent events in a timely manner and can lead to an `OVERFLOW` state where events are lost.
-   **Direct State Modification:** Directly accessing and modifying non-thread-safe game state (e.g., an ArrayList of loaded models) from the consumer callback is a guaranteed race condition and will lead to unpredictable behavior and crashes.

## Data Pipeline

The flow of data is initiated by the operating system and terminates in the application logic provided by the consumer.

> Flow:
> OS File System Event -> Java NIO WatchService -> **PathWatcherThread** `run()` loop -> BiConsumer Callback -> Application Logic (e.g., Asset Reload Task)

