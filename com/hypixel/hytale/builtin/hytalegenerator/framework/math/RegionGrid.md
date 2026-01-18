---
description: Architectural reference for RegionGrid
---

# RegionGrid

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Transient

## Definition
```java
// Signature
public class RegionGrid {
```

## Architecture & Concepts
The RegionGrid is a fundamental mathematical utility within the world generation framework. It imposes a coarse-grained, two-dimensional grid over the world's chunk coordinate system. Its primary function is to translate any given chunk coordinate into the boundaries of the "region" that contains it.

This abstraction is critical for systems that must operate on large, contiguous areas of the world rather than individual chunks. Key use cases include:

-   **Data Persistence:** Grouping chunks into region files for efficient disk I/O.
-   **Procedural Generation:** Scoping large-scale generation passes (e.g., biome placement, river generation) to a deterministic grid.
-   **Level of Detail (LOD):** Managing the loading and unloading of large world areas based on region visibility.

RegionGrid is a pure, stateless computational tool. It contains no game logic, I/O handles, or references to world data. It exclusively provides the spatial reasoning required by higher-level world management systems.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand via its public constructor. It is not managed by a dependency injection framework or service locator. Typically, a world generator or a spatial partitioning system will create a RegionGrid instance with a fixed size for the duration of a task.
-   **Scope:** The lifetime of a RegionGrid object is typically short and bound to the scope of the method or task that requires it. For example, a world-saving process might create one, use it to iterate over all regions, and then discard it.
-   **Destruction:** As a simple Java object with no external resources, it is automatically reclaimed by the garbage collector once all references to it are dropped. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** The object is **immutable**. Its internal state, consisting of regionSizeX and regionSizeZ, is set once during construction and can never be modified.
-   **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that multiple threads can invoke its methods concurrently without any risk of data corruption or inconsistent reads. No external locking or synchronization is required when sharing a RegionGrid instance across threads.

## API Surface
The public API provides methods to calculate the bounding box of a region in the chunk coordinate system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| regionMinX(int chunkX) | int | O(1) | Calculates the minimum chunk X-coordinate for the region containing the given chunk. |
| regionMinZ(int chunkZ) | int | O(1) | Calculates the minimum chunk Z-coordinate for the region containing the given chunk. |
| regionMaxX(int chunkX) | int | O(1) | Calculates the maximum (exclusive) chunk X-coordinate for the region. |
| regionMaxZ(int chunkZ) | int | O(1) | Calculates the maximum (exclusive) chunk Z-coordinate for the region. |

## Integration Patterns

### Standard Usage
A RegionGrid should be instantiated once with a fixed size and reused for all calculations related to that grid structure.

```java
// Define a standard region size of 32x32 chunks
final int REGION_WIDTH_IN_CHUNKS = 32;
RegionGrid regionGrid = new RegionGrid(REGION_WIDTH_IN_CHUNKS, REGION_WIDTH_IN_CHUNKS);

// A system receives a request to process a chunk at a specific coordinate
int targetChunkX = 127;
int targetChunkZ = -15;

// Use the grid to determine the boundaries of the entire region to load or generate
int regionStartX = regionGrid.regionMinX(targetChunkX); // Result: 96
int regionStartZ = regionGrid.regionMinZ(targetChunkZ); // Result: -32

// The system can now operate on all chunks from (96, -32) to (128, 0)
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in Loops:** Avoid creating a new RegionGrid inside a tight loop that processes many chunks. This is wasteful and generates unnecessary garbage. If the region size is constant for the task, create one instance and reuse it.
-   **Stateful Wrapping:** Do not wrap this class in a mutable container or subclass it to add state. Its value and safety derive directly from its immutability.

## Data Pipeline
RegionGrid is not a pipeline stage itself but a computational tool used *within* a stage to transform spatial data. It acts as a coordinate system translator, enabling systems to switch from chunk-based addressing to region-based addressing.

> Flow:
> Chunk-based Task -> **RegionGrid (Coordinate Translation)** -> Region-based Task (e.g., Region File I/O)

