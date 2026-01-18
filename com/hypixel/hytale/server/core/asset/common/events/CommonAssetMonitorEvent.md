---
description: Architectural reference for CommonAssetMonitorEvent
---

# CommonAssetMonitorEvent

**Package:** com.hypixel.hytale.server.core.asset.common.events
**Type:** Transient

## Definition
```java
// Signature
public class CommonAssetMonitorEvent extends AssetMonitorEvent<Void> {
```

## Architecture & Concepts
The CommonAssetMonitorEvent is a fundamental data transfer object within the server's asset hot-reloading pipeline. It functions as an immutable message, encapsulating a set of file system changes that have occurred within a specific asset pack.

This event is produced by a low-level file system watcher, the AssetMonitor, and broadcast to the rest of the engine via an event bus. Its primary role is to decouple the act of detecting file changes from the logic of processing those changes. Systems like the AssetManager or live-reloading services subscribe to this event to trigger asset invalidation, recompilation, or client-side updates.

The use of the generic type **Void** signifies that this event's contract is purely informational; it reports *which* paths have changed, but does not carry the new asset data itself. Consumers are responsible for subsequently reading the modified files from the disk.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the core AssetMonitor service in response to native file system notifications (e.g., create, modify, delete).
-   **Scope:** Short-lived and transactional. An instance exists only for the duration of its propagation through the event bus to all registered subscribers. It holds no persistent state.
-   **Destruction:** Becomes eligible for garbage collection immediately after the event bus has finished dispatching it. There is no manual cleanup or pooling mechanism for this object.

## Internal State & Concurrency
-   **State:** **Immutable**. Once a CommonAssetMonitorEvent is constructed, its internal lists of file paths are considered final. This guarantees that all consumers receive the exact same snapshot of the file system changes.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, an instance can be safely published from a file-watcher thread and consumed by multiple worker threads without requiring any locks or synchronization primitives.

## API Surface
The public API is inherited entirely from its parent, AssetMonitorEvent. The constructor is the only symbol defined directly on the class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetPack() | String | O(1) | Returns the identifier of the asset pack where changes occurred. |
| getCreatedOrModified() | List<Path> | O(1) | Returns an immutable view of files that were created or modified. |
| getRemoved() | List<Path> | O(1) | Returns an immutable view of files that were deleted. |
| getCreatedOrModifiedDirectories() | List<Path> | O(1) | Returns an immutable view of directories that were created or modified. |
| getRemovedDirectories() | List<Path> | O(1) | Returns an immutable view of directories that were deleted. |

## Integration Patterns

### Standard Usage
The canonical interaction pattern is to create a listener method within a service and register it with the engine's central EventBus. The method should be annotated to subscribe to the CommonAssetMonitorEvent type.

```java
// In an asset processing service
@Subscribe
public void handleAssetChanges(CommonAssetMonitorEvent event) {
    // WARNING: Avoid long-running operations on the event bus thread.
    // Offload heavy processing to a dedicated worker pool.
    log.info("File system changes detected in asset pack: " + event.getAssetPack());

    for (Path modifiedFile : event.getCreatedOrModified()) {
        assetReloadService.scheduleReload(modifiedFile);
    }

    for (Path removedFile : event.getRemoved()) {
        assetCache.invalidate(removedFile);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Never create an instance of this event using its constructor. Doing so can inject false file system events into the engine, leading to state desynchronization, unnecessary asset reloading, or cache corruption. These events must only originate from the authoritative AssetMonitor service.
-   **State Modification:** Do not attempt to modify the lists returned by the getter methods. While the method signature returns a List, the underlying implementation is intended to be immutable. Attempting to call methods like add or remove will likely throw an UnsupportedOperationException and violates the object's design contract.

## Data Pipeline
The flow of this event through the system is linear and unidirectional, forming a critical link in the live-development feedback loop.

> Flow:
> OS File System Notification -> AssetMonitor Service -> **CommonAssetMonitorEvent** (Instantiation) -> Engine Event Bus -> Subscribed Services (e.g., AssetManager, LiveReloader) -> Asset Cache Invalidation / Data Reload

