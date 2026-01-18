---
description: Architectural reference for NSimplePixelBuffer
---

# NSimplePixelBuffer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Transient

## Definition
```java
// Signature
public class NSimplePixelBuffer<T> extends NPixelBuffer<T> {
```

## Architecture & Concepts
NSimplePixelBuffer is a specialized, memory-optimized data structure for storing a 3D grid of voxel data, referred to as pixels. It is a fundamental component within the world generation system, designed to represent a fixed-size chunk of the world (SIZE x SIZE x SIZE).

The core architectural concept is its state-based internal representation, which provides a significant performance and memory advantage when handling large, homogeneous regions of voxels (e.g., air, water, stone). Instead of immediately allocating a full 3D array, the buffer transitions through several states:

1.  **EMPTY:** The initial state. The buffer contains no data and consumes minimal memory.
2.  **SINGLE_VALUE:** If the entire buffer is filled with a single, uniform value, it stores only that one value. This is extremely efficient for representing large volumes of a single block type.
3.  **ARRAY:** Only when a second, different value is introduced does the buffer transition to this state. It allocates a full internal array and populates it, paying the full memory and performance cost.

This "lazy allocation" strategy is critical for the performance of the world generator, as many chunks begin as completely solid or completely empty.

## Lifecycle & Ownership
-   **Creation:** NSimplePixelBuffer instances are created on-demand by world generation processes. They are typically instantiated by higher-level managers, such as a BufferBundle, which orchestrates the data flow for a specific region of the world.
-   **Scope:** The lifetime of an instance is short and tied directly to a specific generation task. It is created, populated by one or more generation passes, read by subsequent passes, and then becomes eligible for garbage collection. It does not persist beyond the scope of the generation job.
-   **Destruction:** There is no explicit destruction or cleanup method. Ownership is managed by the Java Garbage Collector. Once all references from the originating generation task are released, the object and its internal array (if allocated) are reclaimed.

## Internal State & Concurrency
-   **State:** The internal data representation is highly mutable and is governed by the private `state` enum. The primary purpose of the class's logic, particularly in `setPixelContent`, is to manage the transitions between these states. The buffer's memory footprint changes dramatically based on its current state.

-   **Thread Safety:** This class is **not thread-safe**. All public methods that modify internal state, such as `setPixelContent` and `copyFrom`, perform non-atomic check-then-act operations on the `state` field. Concurrent writes from multiple threads will lead to race conditions, resulting in data corruption, inconsistent state transitions, or `NullPointerException` if one thread nullifies a field while another is attempting to access it.

    **WARNING:** Instances of NSimplePixelBuffer must only be accessed and modified by a single thread at a time. Any multi-threaded generation system must ensure that each buffer instance is confined to a single worker thread or that external locking mechanisms are implemented.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPixelContent(position) | T | O(1) | Retrieves the pixel content at the given 3D position. Performance is constant but slightly faster in the SINGLE_VALUE state. |
| setPixelContent(position, value) | void | O(1) / O(N) | Sets the pixel content. This is the most critical method. It is O(1) for most calls, but triggers an O(N) operation (where N is total buffer size) on the first call that transitions the state from SINGLE_VALUE to ARRAY. |
| copyFrom(sourceBuffer) | void | O(N) | Performs a deep copy of the source buffer's contents. Complexity depends on the source's state, being O(N) if the source is in the ARRAY state. |
| getMemoryUsage() | MemInstrument.Report | O(1) | Reports the estimated memory consumption of the buffer, which varies significantly based on its internal state. |

## Integration Patterns

### Standard Usage
The intended use is within a sequential world generation pipeline. A generator pass creates a buffer, often filling it with a single value, and subsequent passes modify it to add detail.

```java
// A generator pass receives a buffer, likely pre-filled with a base material.
// NSimplePixelBuffer<Block> terrainBuffer = ...;

// A subsequent pass carves a cave into the buffer.
// This first call might trigger the transition to the ARRAY state.
Vector3i cavePosition = new Vector3i(10, 20, 5);
terrainBuffer.setPixelContent(cavePosition, Block.AIR);

// Further modifications are now made to the internal array.
terrainBuffer.setPixelContent(cavePosition.add(0, 1, 0), Block.AIR);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Accessing and modifying a single NSimplePixelBuffer instance from multiple threads without external synchronization is unsafe and will lead to unpredictable behavior and data corruption.
-   **Premature Allocation:** Filling the buffer with random or noisy data immediately will defeat its core optimization, as it will force an immediate transition to the expensive ARRAY state. Generation should be ordered from large-scale, uniform features to small-scale, detailed features.
-   **Incorrect State Assumption:** Do not write code that assumes the internal state of the buffer. Always interact with it through its public API; the implementation detail of whether it is a single value or an array should be considered an opaque optimization.

## Data Pipeline
NSimplePixelBuffer acts as a data container, or a "stage," for voxel data as it is processed by the world generator. It is not a processing node itself but rather the artifact that is passed between them.

> Flow:
> Base Terrain Pass -> **NSimplePixelBuffer** (Write: Filled with Stone) -> Cave Generation Pass (Read/Write: Carves Air) -> Ore Generation Pass (Read/Write: Places Ores) -> Voxel Mesher (Read) -> Renderable Chunk Mesh

