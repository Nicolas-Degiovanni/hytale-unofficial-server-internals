---
description: Architectural reference for PathEvent
---

# PathEvent

**Package:** com.hypixel.hytale.server.core.asset.monitor
**Type:** Transient

## Definition
```java
// Signature
public class PathEvent {
```

## Architecture & Concepts
The PathEvent class is a simple, immutable data transfer object (DTO) that represents a single, discrete event occurring on the filesystem. It serves as the fundamental unit of communication within the server's asset monitoring and hot-reloading subsystem.

Its primary architectural purpose is to decouple the low-level filesystem watching mechanism (e.g., the operating system's native file notification API) from the higher-level asset management logic. By encapsulating a filesystem change into a well-defined, timestamped object, the system can process asset updates in a structured, reliable, and testable manner. It acts as a message in an event-driven pipeline, signaling that a specific path was affected at a specific time.

## Lifecycle & Ownership
- **Creation:** PathEvent instances are exclusively instantiated by the core asset monitoring service, likely a class such as AssetMonitor. This occurs in direct response to a notification received from the underlying `java.nio.file.WatchService`.
- **Scope:** The lifecycle of a PathEvent is extremely short and ephemeral. It is created, placed into an event queue or passed to a handler, and becomes eligible for garbage collection immediately after processing. It does not persist beyond the scope of a single event handling cycle.
- **Destruction:** The object is managed by the Java Garbage Collector. Once all references from the event queue and processing logic are cleared, it is destroyed. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal fields `eventKind` and `timestamp` are set once during construction and cannot be modified thereafter. This design guarantees that an event's data is consistent and cannot be accidentally altered during its processing lifetime.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. An instance of PathEvent can be safely shared and read across multiple threads without any need for external synchronization or locking. This is critical for the asset monitoring system, which operates on a background thread while delivering events to the main game thread.

## API Surface
The public contract is minimal, providing read-only access to the event's data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEventKind() | EventKind | O(1) | Returns the type of filesystem event (e.g., CREATED, MODIFIED, DELETED). |
| getTimestamp() | long | O(1) | Returns the system timestamp (in milliseconds) when the event was created. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by typical game logic developers. It is an internal component consumed by the asset management system. A consumer would typically receive this object from a queue or callback.

```java
// Hypothetical consumer within the asset system
void processAssetEvents(Queue<PathEvent> eventQueue) {
    PathEvent event = eventQueue.poll();
    if (event != null) {
        if (event.getEventKind() == EventKind.MODIFIED) {
            // Trigger asset reload logic
            assetReloader.handleModification(event);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a PathEvent using `new PathEvent()`. These events must originate from the authoritative filesystem monitoring service to ensure they represent actual disk state changes. Manual creation can lead to a desynchronized asset state.
- **Timestamp Reliance for Ordering:** Do not rely solely on the timestamp for strict event ordering between different paths, as filesystem notifications can sometimes be delivered out of order by the OS. The system should be robust against minor timing discrepancies.

## Data Pipeline
PathEvent is a message that flows through the asset hot-reloading data pipeline. It translates a raw OS-level signal into a structured piece of data for the engine.

> Flow:
> OS Filesystem Change -> `java.nio.file.WatchService` -> AssetMonitor Service -> **PathEvent Instantiation** -> Concurrent Event Queue -> Asset Reloading Subsystem

