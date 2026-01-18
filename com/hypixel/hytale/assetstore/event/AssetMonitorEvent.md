---
description: Architectural reference for AssetMonitorEvent
---

# AssetMonitorEvent

**Package:** com.hypixel.hytale.assetstore.event
**Type:** Transient Data Model

## Definition
```java
// Signature
public abstract class AssetMonitorEvent<T> implements IEvent<T> {
```

## Architecture & Concepts
The AssetMonitorEvent is a foundational data structure within the engine's asset hot-reloading and live-editing framework. It is not a service or manager, but rather an immutable message that represents a batch of detected file system changes for a specific asset pack.

This class acts as a data contract between the low-level file system monitoring service and higher-level asset management systems. By implementing the IEvent interface, it is designed to be dispatched onto a central Event Bus. This decouples the source of the file system notification from the consumers that need to react to it, such as the TextureManager, SoundManager, or ModelManager.

The primary design goal is to bundle related file system operations (creations, modifications, deletions) into a single, atomic event. This prevents event storms and allows consuming systems to process changes in a coherent transaction, ensuring a consistent state.

### Lifecycle & Ownership
- **Creation:** An instance of a concrete AssetMonitorEvent subclass is created by the engine's core file system watcher service. This occurs when the watcher detects and aggregates a set of file or directory changes within a monitored asset pack directory.
- **Scope:** Extremely short-lived. The event object's lifetime is bound to the duration of its dispatch on the event bus. It is created, broadcast to all subscribers, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no manual cleanup or resource release methods. Subscribers must not retain references to the event object after their handler method completes.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are declared final and are initialized only once during construction. The object's state cannot be changed after it has been created, which is critical for safe, predictable processing in a multi-threaded eventing environment.

- **Thread Safety:** **Conditionally Thread-Safe**. The object itself is immutable and therefore safe to read from multiple threads simultaneously. However, its thread safety is contingent on the behavior of the code that creates it. The lists passed into the constructor are not defensively copied.

    **WARNING:** The system that creates this event **must not** modify the underlying lists after passing them to the constructor. Doing so would violate the immutability contract and lead to severe concurrency bugs.

## API Surface
The public API is composed entirely of non-blocking accessor methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetPack() | String | O(1) | Returns the identifier of the asset pack where changes occurred. |
| getCreatedOrModifiedFilesToLoad() | List<Path> | O(1) | Returns a list of files that were newly created or modified. |
| getRemovedFilesToUnload() | List<Path> | O(1) | Returns a list of files that were removed. |
| getCreatedOrModifiedDirectories() | List<Path> | O(1) | Returns a list of directories that were newly created or modified. |
| getRemovedFilesAndDirectories() | List<Path> | O(1) | Returns a list of all files and directories that were removed. |

## Integration Patterns

### Standard Usage
This class is abstract and not intended to be instantiated directly. Developers will interact with it by subscribing to one of its concrete subclasses on the event bus. The typical pattern is to receive the event as a method parameter in a registered event handler.

```java
// Example of a subscriber service reacting to asset changes
// Note: A concrete event type like 'GameAssetMonitorEvent' would be used.

@Subscribe
public void onAssetFilesystemUpdate(GameAssetMonitorEvent event) {
    // Process files that need to be loaded or reloaded
    for (Path assetPath : event.getCreatedOrModifiedFilesToLoad()) {
        assetLoaderService.scheduleReload(event.getAssetPack(), assetPath);
    }

    // Process files that need to be unloaded from memory
    for (Path assetPath : event.getRemovedFilesToUnload()) {
        assetLoaderService.scheduleUnload(event.getAssetPack(), assetPath);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Retaining References:** Do not store the event object in a field or collection for later processing. It is a transient message representing a point-in-time snapshot. Caching it can lead to processing stale data.
- **Modifying Backing Lists:** The service that creates this event must treat the lists it passes to the constructor as relinquished. Any subsequent modification to those lists will cause undefined behavior in downstream subscribers.
- **Direct Instantiation:** As an abstract class, `new AssetMonitorEvent(...)` is not possible. Do not attempt to create your own subclasses unless you are extending the core asset monitoring system itself.

## Data Pipeline
The AssetMonitorEvent is a critical link in the data flow for live asset updates. It transforms low-level file system signals into a structured, actionable message for the rest of the engine.

> Flow:
> File System I/O Notification -> File Watcher Service -> **AssetMonitorEvent (Creation)** -> Event Bus Dispatch -> AssetManager/Subscribers (Consumption) -> Asset Load/Unload Task Scheduling

