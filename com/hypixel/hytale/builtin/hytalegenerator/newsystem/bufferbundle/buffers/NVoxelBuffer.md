---
description: Architectural reference for NVoxelBuffer
---

# NVoxelBuffer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class NVoxelBuffer<T> extends NBuffer {
```

## Architecture & Concepts

The NVoxelBuffer is a specialized, memory-optimized data structure designed to hold a fixed-size 8x8x8 volume of generic voxel data. It is a fundamental component within the world generation system, engineered to dramatically reduce the memory footprint required during the creation of world data.

Its core architectural pattern is a state machine that dynamically changes its internal data representation based on the contents of the voxel volume. This allows the system to avoid allocating large arrays for regions of the world that are uniform, such as chunks of solid stone or empty air.

The buffer operates in one of four primary states:

*   **EMPTY:** The initial state, representing a completely empty volume with no data. This is the most memory-efficient state.
*   **SINGLE_VALUE:** Represents a volume where every one of the 512 voxels is identical. Instead of a 512-element array, the buffer stores only a single reference to the voxel type. This is a significant optimization for large, uniform regions.
*   **ARRAY:** The traditional representation. When the volume contains two or more different voxel types, the buffer transitions to this state, allocating a full 512-element array to store each voxel individually. This is the least memory-efficient but most flexible state.
*   **REFERENCE:** A copy-on-write optimization. In this state, the buffer does not hold its own data but instead points to another NVoxelBuffer. This allows multiple logical buffers to share the same underlying data until one of them is modified. Upon modification, the buffer is "dereferenced," creating its own copy of the data to mutate.

This state-based design is critical for performance and memory management in a large-scale procedural generation environment.

### Lifecycle & Ownership
-   **Creation:** An NVoxelBuffer is instantiated directly via its constructor, `new NVoxelBuffer<>(voxelType)`, typically by a higher-level world generation controller or a chunk data manager.
-   **Scope:** The object is transient and its lifetime is typically scoped to a specific generation task. It holds intermediate data for a small section of the world before that data is consumed by other systems, such as a mesher or serializer.
-   **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup methods. It becomes eligible for garbage collection once all references to it are dropped by the generation pipeline.

## Internal State & Concurrency
-   **State:** The NVoxelBuffer is highly mutable. Its primary purpose is to transition between internal representations (single value, full array, or reference) in response to `setVoxelContent` calls. The internal `state` field, along with `singleValue`, `arrayContents`, and `referenceBuffer`, constitutes its entire state.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. All internal operations, particularly `setVoxelContent`, perform non-atomic check-then-act sequences on the `state` field. For example, a write operation might check if the state is `SINGLE_VALUE` and then begin the expensive process of converting it to `ARRAY`. Concurrent writes would lead to race conditions, data corruption, and unpredictable behavior. All access to a given NVoxelBuffer instance **must** be externally synchronized or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getVoxelContent(position) | T | O(k) | Retrieves the voxel at the given position. Complexity is O(k) where k is the depth of the reference chain in the `REFERENCE` state. |
| setVoxelContent(position, value) | void | Amortized O(1) | Sets the voxel at the given position. May trigger an expensive state transition (e.g., from `SINGLE_VALUE` to `ARRAY`), resulting in O(N) complexity where N is 512. |
| reference(sourceBuffer) | void | O(k) | Sets this buffer to be a reference to the `sourceBuffer`. Traverses the source's reference chain to point to the final concrete buffer. |
| getVoxelType() | Class<T> | O(1) | Returns the generic type of the voxels this buffer is configured to hold. |
| getMemoryUsage() | Report | O(1) | Provides a report on the estimated memory consumption of the buffer in its current state. |

## Integration Patterns

### Standard Usage
The NVoxelBuffer is intended to be used by generation algorithms to build up a small volume of the world. The algorithm populates the buffer, which is then passed to a consumer.

```java
// A generator creates and populates a buffer for a specific voxel type.
NVoxelBuffer<Block> blockBuffer = new NVoxelBuffer<>(Block.class);

// Populate the buffer. The first write transitions it from EMPTY to SINGLE_VALUE.
blockBuffer.setVoxelContent(new Vector3i(0, 0, 0), Blocks.STONE);

// A subsequent, different write transitions it from SINGLE_VALUE to ARRAY.
blockBuffer.setVoxelContent(new Vector3i(0, 1, 0), Blocks.DIRT);

// Another system can now read the final data.
Block block = blockBuffer.getVoxelContent(new Vector3i(0, 1, 0));
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never access or modify a single NVoxelBuffer instance from multiple threads without external locking. The internal state transitions are not atomic and will break.
-   **Recursive Referencing:** Do not create circular references (e.g., buffer A references B, and B references A). While the `lastReference` method attempts to flatten chains on creation, a circular dependency will cause a `StackOverflowError` during traversal.
-   **Ignoring State Transition Costs:** Be aware that the first write that breaks uniformity in a `SINGLE_VALUE` buffer will incur a significant performance penalty as it allocates and fills a 512-element array. For performance-critical code, batching writes or pre-allocating as an `ARRAY` might be preferable if the final state is known to be heterogeneous.

## Data Pipeline
The NVoxelBuffer acts as a container or staging area for data, not as a processing node in a pipeline.

> Flow:
> World Generation Algorithm -> `setVoxelContent` -> **NVoxelBuffer (State Management)** -> `getVoxelContent` -> Chunk Mesher or World Serializer

