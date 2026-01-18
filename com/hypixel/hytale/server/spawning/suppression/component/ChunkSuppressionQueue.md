---
description: Architectural reference for ChunkSuppressionQueue
---

# ChunkSuppressionQueue

**Package:** com.hypixel.hytale.server.spawning.suppression.component
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class ChunkSuppressionQueue implements Resource<ChunkStore> {
```

## Architecture & Concepts
The ChunkSuppressionQueue is a transactional data structure that acts as a command buffer for chunk-level spawn suppression changes. It is not a global queue, but rather a `Resource` component that collects requests to enable or disable mob spawning within specific chunks.

Its primary architectural role is to decouple the *request* for a state change from the *execution* of that change. Systems throughout the server can queue operations into a ChunkSuppressionQueue without needing direct access to the core spawning system or worrying about the immediate state of a chunk.

At a designated phase of the server tick, a central processing system within the SpawningPlugin consumes these queues, applying the requested changes in a batched, atomic manner. This pattern prevents race conditions, ensures a consistent world state for spawning calculations within a single tick, and improves performance by consolidating state modifications.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly. They are managed by the engine's resource and component system. A system that needs to alter spawn suppression (e.g., a system managing player-built structures) would typically acquire or create this component on a relevant entity.
-   **Scope:** Extremely short-lived. A ChunkSuppressionQueue is intended to exist for less than a single game tick. It is populated with commands, processed by the spawning system, and then discarded.
-   **Destruction:** The instance is garbage collected after the consuming system has processed its contents at the end of a tick phase. Holding a reference to a queue across multiple ticks is an error and will lead to stale data.

## Internal State & Concurrency
-   **State:** Highly mutable. The object's sole purpose is to accumulate state changes in its internal `toAdd` and `toRemove` lists. It is a stateful container by design.
-   **Thread Safety:** This class is **not thread-safe**. The internal collections are standard, unsynchronized lists. All interactions, both populating the queue and processing it, must occur on the main server thread to prevent data corruption and `ConcurrentModificationException`.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | ResourceType | O(1) | **Static**. Retrieves the registered type definition for this resource from the SpawningPlugin. |
| getToAdd() | List | O(1) | Returns the internal list of pending suppression additions. Warning: Modifying this list directly is unsupported. |
| getToRemove() | List | O(1) | Returns the internal list of pending suppression removals. Warning: Modifying this list directly is unsupported. |
| queueForAdd(ref, entry) | void | O(1) | Enqueues a command to apply a ChunkSuppressionEntry to the referenced ChunkStore. |
| queueForRemove(ref) | void | O(1) | Enqueues a command to remove spawn suppression from the referenced ChunkStore. |
| clone() | Resource | O(N) | Creates a shallow copy of the queue, where N is the total number of queued operations. |

## Integration Patterns

### Standard Usage
The intended use is to acquire an instance from the component system, populate it with desired changes, and allow the engine to process it automatically.

```java
// A system responsible for world changes obtains the queue
// (Note: The exact acquisition mechanism may vary)
ChunkSuppressionQueue queue = world.getResource(ChunkSuppressionQueue.getResourceType());

// Game logic determines a chunk needs spawn suppression
Ref<ChunkStore> targetChunk = getChunkRefAt(position);
ChunkSuppressionEntry suppressionData = new ChunkSuppressionEntry(...);

// Queue the operation for processing later in the tick
queue.queueForAdd(targetChunk, suppressionData);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ChunkSuppressionQueue()`. The component system manages its lifecycle. Unmanaged instances will not be processed by the spawning system.
-   **Cross-Tick References:** Do not cache and reuse a ChunkSuppressionQueue across multiple server ticks. It is a single-use, transactional object.
-   **Concurrent Modification:** Do not access or modify a queue from an asynchronous task or worker thread. All operations must be synchronized with the main server tick.

## Data Pipeline
The ChunkSuppressionQueue is a temporary buffer in a larger data flow that modifies the persistent state of the world's chunks.

> Flow:
> Game Event (e.g., player action) -> Game Logic System -> **ChunkSuppressionQueue**.queueForAdd() -> Spawning System (end of tick) -> Read Queue -> Modify ChunkStore State -> Clear/Discard Queue

