---
description: Architectural reference for AssetMonitorHandler
---

# AssetMonitorHandler

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface AssetMonitorHandler extends BiPredicate<Path, EventKind>, Consumer<Map<Path, EventKind>> {
```

## Architecture & Concepts
The AssetMonitorHandler interface defines a behavioral contract for components that need to react to file system changes, forming the core of the server's asset hot-reloading system. It is a specialized functional interface that elegantly combines two distinct responsibilities using standard Java functional types:

1.  **Filtering (BiPredicate):** It determines *if* a file system event for a given path is relevant to the handler. This allows the central AssetMonitor service to efficiently route events only to interested listeners.
2.  **Processing (Consumer):** It defines the action to be taken when one or more relevant file system events have occurred. This is the callback that triggers the actual asset reload or reprocessing logic.

By implementing this interface, a system (e.g., a script manager, a configuration loader) can subscribe to live changes in the underlying asset files without needing to know the low-level details of file system watching. The `getKey` method provides a stable identity, which is critical for the managing service to register, unregister, and prevent duplicate handlers for the same logical resource.

## Lifecycle & Ownership
As an interface, AssetMonitorHandler itself has no lifecycle. The lifecycle pertains to its concrete implementations.

-   **Creation:** An implementation of AssetMonitorHandler is typically instantiated by a service that manages a specific type of asset. For example, a `WorldScriptManager` would create a handler instance to monitor its script directory when a world is loaded.
-   **Scope:** The lifetime of a handler is directly tied to the lifetime of the resource it monitors. It is registered with the global `AssetMonitor` service upon creation and persists as long as its parent system is active.
-   **Destruction:** The handler is unregistered from the `AssetMonitor` service when the resource it was watching is no longer active (e.g., a world is unloaded). The parent system is responsible for unregistering the handler to prevent memory leaks and unnecessary file system monitoring. Failure to unregister is a critical resource leak.

## Internal State & Concurrency
-   **State:** The interface is stateless. However, concrete implementations are almost always stateful, holding references to the assets or systems they manage.
-   **Thread Safety:** **CRITICAL WARNING:** Implementations of this interface are not guaranteed to be invoked on the main server thread. The file system watching service typically operates on a dedicated background thread. Therefore, all implementations of the `accept` method **must be thread-safe**. Any interaction with non-thread-safe game state (e.g., world objects, entities) must be marshaled to the main game loop via a task scheduler or concurrent queue. Direct modification of game state from within the handler will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is composed of methods inherited from its parent interfaces and one method for identity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Path, EventKind) | boolean | O(1) | Inherited from BiPredicate. Invoked by the AssetMonitor to filter events. Should return true if the event is relevant. |
| accept(Map<Path, EventKind>) | void | O(N) | Inherited from Consumer. Invoked with a batch of relevant events. Contains the core hot-reloading logic. |
| getKey() | Object | O(1) | Returns a unique, stable identifier for this handler instance. Used for registration and equality checks. |

## Integration Patterns

### Standard Usage
A managing service creates and registers a handler to receive callbacks for a specific asset directory. The logic is self-contained within the handler implementation.

```java
// Example of a handler implementation and registration
AssetMonitor assetMonitor = context.getService(AssetMonitor.class);

// 1. Create the handler, often as an anonymous class or lambda
AssetMonitorHandler scriptHandler = new WorldScriptHandler(world);

// 2. Register it with the central monitoring service
assetMonitor.register(scriptHandler);

// When the world is unloaded:
// assetMonitor.unregister(scriptHandler.getKey());
```

### Anti-Patterns (Do NOT do this)
-   **Blocking Operations:** Do not perform long-running or blocking operations (e.g., network requests, heavy disk I/O) directly within the `accept` method. This will stall the entire asset monitoring thread, delaying or dropping file change notifications for all other systems. Offload heavy work to a separate worker thread pool.
-   **Unstable Key:** Do not implement `getKey` to return a new object on each call. The key must be stable and have a correct `equals` and `hashCode` implementation for the registration map to function correctly.
-   **Assuming Main Thread:** Never assume `accept` is called on the main server thread. Always dispatch game state mutations to the appropriate thread.

## Data Pipeline
The handler acts as a filter and sink in the asset event pipeline.

> Flow:
> File System Event (Create/Modify/Delete) -> OS Notification -> Java WatchService -> **AssetMonitor** (Central Dispatcher) -> **AssetMonitorHandler.test()** (Filter) -> **AssetMonitorHandler.accept()** (Action) -> Game State Update (via Task Scheduler)

