---
description: Architectural reference for Indexer
---

# Indexer

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Transient State Object

## Definition
```java
// Signature
public class Indexer {
```

## Architecture & Concepts
The Indexer is a fundamental utility component used for mapping object instances to unique, sequential integer identifiers. Its primary role is to act as a transient, in-memory registry or palette during data processing tasks, most notably within world generation pipelines.

In a system like world generation, complex objects such as Biomes, Block States, or other metadata need to be stored in a compact format, like a chunk's data array. The Indexer provides the mechanism to convert these object references into small integers. Each unique object is assigned an ID the first time it is processed, and subsequent encounters with the same object will return the same, previously assigned ID.

This component is not a global service but rather a stateful, task-specific tool. A new Indexer is typically created for a discrete unit of work, such as building a biome palette for a world region, ensuring that the generated IDs are consistent and scoped only to that specific task.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new Indexer()`) by a higher-level system that requires object-to-integer mapping. For example, a WorldGenerator might create an Indexer at the start of a generation pass.
- **Scope:** The lifecycle of an Indexer is strictly bound to the parent object that creates it. It persists only for the duration of the specific task it was created for.
- **Destruction:** The object is eligible for garbage collection as soon as the reference held by its owner is released. It holds no static references and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The core state is a mutable HashMap that stores the mapping from an Object to its assigned Integer ID. The state grows dynamically as new, unique objects are processed by the getIdFor method. The internal state is entirely dependent on the history and sequence of objects it has indexed.

- **Thread Safety:** **This class is not thread-safe.** The internal HashMap is not synchronized. Concurrent invocations of getIdFor from multiple threads on a single Indexer instance will result in a race condition. The use of `computeIfAbsent` is atomic for the map itself, but the lambda `k -> this.ids.size()` is not an atomic part of the check. This can lead to corrupted state, non-unique IDs, and unpredictable behavior under concurrent load.

    **WARNING:** An Indexer instance must be confined to a single thread of execution. If multi-threaded access is unavoidable, it must be protected by external synchronization mechanisms (e.g., a `synchronized` block).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getIdFor(Object o) | int | Amortized O(1) | Returns the integer ID for the given object. If the object has not been seen before, it is assigned a new, unique ID based on the current size of the index, and that ID is returned. |
| size() | int | O(1) | Returns the total number of unique objects currently held in the index. This corresponds to the next ID that will be assigned. |

## Integration Patterns

### Standard Usage
The Indexer is designed to be created and used within the scope of a single, well-defined process. The owning class is responsible for its lifecycle.

```java
// Example: Building a palette for a specific generation task
Indexer blockPalette = new Indexer();

// Process various block types
int stoneId = blockPalette.getIdFor(Block.STONE);      // Returns 0
int dirtId = blockPalette.getIdFor(Block.DIRT);        // Returns 1
int duplicateId = blockPalette.getIdFor(Block.STONE);  // Returns 0

// The size reflects the number of unique items
int uniqueBlockCount = blockPalette.size(); // Value is 2
```

### Anti-Patterns (Do NOT do this)
- **Shared Global Instance:** Do not declare an Indexer as a `static` or global singleton. Its state is context-specific and sharing it across unrelated systems will cause ID collisions and data corruption. Each independent task requires its own Indexer instance.
- **Concurrent Modification:** Do not access a single Indexer instance from multiple threads without external locking. This will lead to race conditions and an inconsistent internal state.

## Data Pipeline
The Indexer functions as a stateful transformer in a data processing pipeline, converting high-level object representations into a primitive, optimized format.

> Flow:
> High-Level Object (e.g., Biome, BlockState) -> **Indexer.getIdFor()** -> Integer ID -> Compact Data Structure (e.g., Chunk Data Array, Network Packet)

