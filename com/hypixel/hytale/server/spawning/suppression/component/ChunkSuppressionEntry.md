---
description: Architectural reference for ChunkSuppressionEntry
---

# ChunkSuppressionEntry

**Package:** com.hypixel.hytale.server.spawning.suppression.component
**Type:** Data Component

## Definition
```java
// Signature
public class ChunkSuppressionEntry implements Component<ChunkStore> {
```

## Architecture & Concepts
The ChunkSuppressionEntry is a passive data component, not an active service. Within Hytale's server architecture, it acts as a data container that attaches spawn suppression rules to a specific ChunkStore. Its sole purpose is to define vertical regions, or *spans*, within a world chunk where mob spawning is either completely disabled or restricted to certain mob roles.

This component is a fundamental part of the server's declarative spawning system. Instead of containing complex logic, it holds immutable state that is queried by higher-level systems, primarily the SpawningPlugin. By attaching this component to a chunk's data store, game logic can effectively "paint" no-spawn zones onto the world grid. This design decouples the *rules* of spawning (defined here) from the *mechanics* of spawning (handled by the SpawningPlugin), promoting a clean, data-oriented architecture.

The inner class, SuppressionSpan, defines a single continuous vertical volume by its minimum and maximum Y coordinates, the unique ID of the object causing the suppression, and an optional set of specific mob roles to suppress.

## Lifecycle & Ownership
- **Creation:** A ChunkSuppressionEntry is instantiated by server-side game logic when a condition for spawn suppression is met within a chunk. Common triggers include a player placing a specific block or a world-generation feature that designates a no-spawn area. It is then attached to the target ChunkStore via its component management interface.

- **Scope:** The lifecycle of a ChunkSuppressionEntry is strictly bound to the ChunkStore it is attached to. It persists as long as the chunk data exists and the suppression condition remains active. It is serialized and saved to disk as part of the chunk's data.

- **Destruction:** The component is marked for garbage collection when it is explicitly removed from the ChunkStore's component map. This typically occurs when the suppressing object or condition is removed from the world.

## Internal State & Concurrency
- **State:** **Immutable**. This is a critical design feature. Upon construction, the list of SuppressionSpan objects is wrapped in an unmodifiable view. The SuppressionSpan itself ensures its internal role set is also unmodifiable. Once an instance is created, its state cannot be altered, preventing entire classes of bugs related to state corruption. To change suppression rules, a new ChunkSuppressionEntry component must be created to replace the old one.

- **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that it can be read concurrently by any number of threads without requiring locks or other synchronization primitives. This is essential for performance, as the spawning system may perform checks across many chunks in parallel worker threads while the main server thread continues with other game logic.

## API Surface
The public API is designed for efficient, read-only queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component, used for registration and lookup. |
| getSuppressionSpans() | List | O(1) | Returns an unmodifiable view of the vertical suppression regions in this chunk. |
| containsOnly(UUID) | boolean | O(1) | Performs a fast check to see if the chunk is suppressed by a single, specific source. |
| isSuppressingRoleAt(int, int) | boolean | O(N) | The primary query method. Determines if a specific mob role is suppressed at a given Y-coordinate. N is the number of spans. |
| clone() | Component | O(1) | Creates a shallow copy. As the internal state is immutable, this is a safe and cheap operation. |

## Integration Patterns

### Standard Usage
The component should always be retrieved from a ChunkStore and then queried. The SpawningPlugin is the canonical consumer of this component.

```java
// Example from within a spawning system
ChunkStore targetChunk = world.getChunkStore(chunkPosition);
ChunkSuppressionEntry suppression = targetChunk.getComponent(ChunkSuppressionEntry.getComponentType());

if (suppression != null) {
    if (suppression.isSuppressingRoleAt(mobRole.getIndex(), yPosition)) {
        // Abort spawning attempt for this location
        return;
    }
}
// Proceed with spawning...
```

### Anti-Patterns (Do NOT do this)
- **Attempting Modification:** Do not attempt to modify the list returned by getSuppressionSpans. The design enforces immutability, and any such attempt will throw an UnsupportedOperationException at runtime.

- **Direct Instantiation for Querying:** Do not use `new ChunkSuppressionEntry()` for any purpose other than attaching it to a ChunkStore. The component is meaningless in isolation; its context is the chunk it belongs to.

- **Assuming Span Order:** The `isSuppressingRoleAt` method iterates through the list of spans. While an implementation might benefit from a sorted list, the public contract does not guarantee any specific order. Code should not rely on spans being sorted by Y-level or any other attribute.

## Data Pipeline
ChunkSuppressionEntry primarily serves as a data source for the spawning system. It does not process data itself.

> **Flow (Spawning Check):**
> Spawning System Tick -> Identify Target Chunk -> `ChunkStore.getComponent()` -> **ChunkSuppressionEntry** -> `isSuppressingRoleAt()` -> Decision (Allow/Deny Spawn)

