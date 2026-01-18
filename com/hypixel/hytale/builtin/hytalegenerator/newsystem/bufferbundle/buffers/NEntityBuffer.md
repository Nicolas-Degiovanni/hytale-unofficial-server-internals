---
description: Architectural reference for NEntityBuffer
---

# NEntityBuffer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class NEntityBuffer extends NBuffer {
```

## Architecture & Concepts
The NEntityBuffer is a specialized, mutable data container designed for the world generation pipeline. Its primary role is to aggregate entity placement instructions—represented by EntityPlacementData objects—that are produced during a single generation task. It acts as a temporary scratchpad or buffer where a generator can record which entities should be spawned and where, without immediately instantiating them in the game world.

This class is a core component of the "BufferBundle" system, which collects various types of generated data (blocks, entities, biomes) into a cohesive unit of work.

A key architectural feature is its support for reference-based copying via the copyFrom method. This allows one NEntityBuffer to act as a lightweight, read-only pointer to another's data. This is a memory optimization pattern used to pass generation results between stages without the overhead of a deep copy. The internal isReference flag tracks this state, signaling that the buffer does not own its underlying data list.

## Lifecycle & Ownership
-   **Creation:** NEntityBuffer instances are created on-demand by world generation algorithms or tasks. They are not managed by a central registry and are intended to be short-lived. A generator responsible for placing features like trees or dungeons would instantiate an NEntityBuffer to store the resulting entity spawn locations.
-   **Scope:** The lifecycle of an NEntityBuffer is tightly coupled to a specific world generation operation, typically for a single chunk or region. It exists only as long as needed to collect the data and pass it to a consuming system.
-   **Destruction:** The object is eligible for garbage collection once the world finalization or population stage has consumed its data and all references to it are dropped. In cases where copyFrom is used, the underlying list of entities will only be garbage collected when both the original buffer and all referencing buffers are out of scope.

## Internal State & Concurrency
-   **State:** The class is highly mutable. Its primary state is the internal list of EntityPlacementData objects, which is lazily instantiated on the first call to addEntity. The isReference boolean flag is also part of its mutable state.
-   **Thread Safety:** **This class is not thread-safe.** The internal use of a standard ArrayList without any synchronization mechanisms makes it vulnerable to race conditions and ConcurrentModificationExceptions if accessed by multiple threads simultaneously. It is designed with the expectation that a single worker thread will own and modify an instance at any given time. External locking must be implemented if concurrent access is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEach(Consumer) | void | O(N) | Iterates over all contained EntityPlacementData objects and applies the given consumer. |
| addEntity(EntityPlacementData) | void | O(1) amortized | Appends a new entity placement instruction to the internal list. Lazily creates the list if it is null. |
| getMemoryUsage() | MemInstrument.Report | O(N) | Calculates and returns an estimate of the memory consumed by the buffer and its contained data. |
| copyFrom(NEntityBuffer) | void | O(1) | Performs a shallow, reference-based copy. The internal list of this buffer will now point to the same list instance as the sourceBuffer. |

## Integration Patterns

### Standard Usage
An NEntityBuffer is typically used as a temporary data accumulator within a single generation step. The generator populates the buffer, which is then passed to a subsequent system for processing.

```java
// A world generator creates and populates the buffer
NEntityBuffer entityBuffer = new NEntityBuffer();
EntityPlacementData treeData = createTreeEntityData(position);
entityBuffer.addEntity(treeData);

// The buffer is passed to a world populator system
worldPopulator.processEntities(entityBuffer);

// The populator then iterates and spawns the entities
public void processEntities(NEntityBuffer buffer) {
    buffer.forEach(placementData -> {
        world.spawnEntity(placementData);
    });
}
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call addEntity from one thread while another thread is iterating using forEach on the same instance. This will lead to unpredictable behavior and crashes.
-   **Modification After Reference Copy:** The copyFrom method creates a shared data reference, not a snapshot. Modifying the source buffer after a copy will affect all referencing buffers. This can be a source of subtle and hard-to-trace bugs.

    ```java
    // DANGEROUS: Modifying source after creating a reference
    NEntityBuffer source = new NEntityBuffer();
    source.addEntity(entityA);

    NEntityBuffer reference = new NEntityBuffer();
    reference.copyFrom(source); // reference now "sees" entityA

    source.addEntity(entityB); // DANGER: reference now also sees entityB
    ```
-   **Modifying a Reference Buffer:** A buffer that was populated via copyFrom should be treated as read-only. Calling addEntity on it will modify the original source buffer's list, violating ownership principles.

## Data Pipeline
The NEntityBuffer serves as a temporary holding area for data as it moves from abstract generation logic to concrete world state.

> Flow:
> World Generation Algorithm -> Creates EntityPlacementData -> **NEntityBuffer.addEntity()** -> Buffer is passed to World Populator -> **NEntityBuffer.forEach()** -> Entity Spawning System -> Entity is created in the World
---

