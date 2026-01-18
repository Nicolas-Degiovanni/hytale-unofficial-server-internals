---
description: Architectural reference for NBufferBundle
---

# NBufferBundle

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle
**Type:** Transient

## Definition
```java
// Signature
public class NBufferBundle implements MemInstrument {
```

## Architecture & Concepts
The NBufferBundle serves as a high-level memory management system specifically designed for the Hytale world generator. Its primary function is to provide efficient, on-demand access to large, sparse, three-dimensional data grids, which are fundamental to procedural generation tasks.

Conceptually, the system is a container that manages multiple specialized data caches. It is composed of three core components:

1.  **NBufferBundle**: The top-level object. It acts as a registry for different types of data grids, mapping an NBufferType to a dedicated Grid instance. It is the sole entry point for creating and accessing these grids.

2.  **Grid**: A specialized, homogeneous cache for a single NBufferType (e.g., terrain height, biome ID, block data). The Grid manages a collection of NBuffer objects, which are the raw data containers. It implements a lazy-loading and eviction strategy to manage memory usage. Data is stored and evicted in vertical columns (spanning the full world height) to optimize for common world generation access patterns. When the Grid reaches its capacity, it attempts to destroy the least recently used columns that are not currently locked by an active Access object.

3.  **Access**: A short-lived, leased view into a specific region of a Grid. When a system needs to read or write world data, it requests an Access object for a given bounding box. This action loads the necessary buffer columns into memory (if not already present) and "pins" them, preventing eviction while the Access is open. This lease-based mechanism is the cornerstone of the system's memory safety, ensuring that data is not deallocated while in use.

This architecture effectively abstracts the complexities of memory allocation, caching, and data locality from the individual world generation algorithms.

### Lifecycle & Ownership
-   **Creation:** An NBufferBundle is instantiated by a high-level world generation orchestrator at the beginning of a generation process. It is not a global singleton; its existence is tied to a specific, stateful generation task.
-   **Scope:** The object persists for the entire duration of the world generation pipeline. It accumulates and manages all intermediate data required by the various generation stages.
-   **Destruction:** The NBufferBundle and all its managed buffers become eligible for garbage collection once the world generation process is complete and the orchestrator releases its reference. The explicit `closeAllAccesses` method should be called as part of the cleanup sequence to invalidate any potentially lingering Access objects, though proper usage relies on consumers closing their own Access instances promptly.

## Internal State & Concurrency
-   **State:** The NBufferBundle is a highly stateful and mutable component. Its primary state is the internal map of Grid objects. Each Grid, in turn, maintains a mutable map of buffer columns and a deque to track usage for its eviction policy. The state of the system changes continuously as generation stages request and release Access objects.

-   **Thread Safety:** **CRITICAL WARNING:** This class is **not thread-safe**. The underlying collections (HashMap, ArrayDeque, ArrayList) are not concurrent. All interactions with a single NBufferBundle instance and its subsidiary objects (Grid, Access) **must** be confined to a single thread. Concurrent access will lead to `ConcurrentModificationException`, data corruption, and unpredictable generator behavior. The design assumes a single-threaded, sequential pipeline of generation stages.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| createGrid(type, capacity) | Grid | O(1) | Registers and allocates a new, empty Grid for the specified buffer type with a given capacity. Throws an assertion error if a Grid for that type or its index already exists. |
| createBufferAccess(type, bounds) | Access | O(N) | Creates a leased view into a region of a Grid. This is a potentially expensive operation, as it may trigger the allocation and loading of many buffer columns. N is proportional to the XZ area of the requested bounds. |
| closeAllAccesses() | void | O(M) | Forcefully closes all open Access objects across all Grids. M is the total number of open Access objects. This is primarily a cleanup or error-recovery mechanism. |
| getGrid(type) | Grid | O(1) | Retrieves the Grid instance associated with the given buffer type. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves a strict acquire-use-release cycle for Access objects. A generator stage should request an Access for the region it needs, perform all its work, and then immediately close the Access to release its lock on the memory.

```java
// A world generator obtains the bundle from its context
NBufferBundle bundle = generatorContext.getBufferBundle();

// Define the required data grid if it doesn't exist
Grid heightmapGrid = bundle.createGrid(HEIGHTMAP_BUFFER_TYPE, 1024);

// 1. Acquire an Access object for a specific work area
Bounds3i workArea = new Bounds3i(0, 0, 0, 16, 0, 16);
NBufferBundle.Access heightmapAccess = bundle.createBufferAccess(HEIGHTMAP_BUFFER_TYPE, workArea);

try {
    // 2. Use the Access to get and modify buffers
    NBufferBundle.Access.View view = heightmapAccess.createView();
    for (Vector3i pos : view.getBounds_bufferGrid()) {
        NBuffer buffer = view.getBuffer(pos).buffer();
        // ... perform generation logic on the buffer
    }
} finally {
    // 3. Release the Access object immediately when done
    heightmapAccess.close();
}
```

### Anti-Patterns (Do NOT do this)
-   **Leaking Access Objects:** Failing to call `close()` on an Access object is the most severe anti-pattern. An open Access prevents the underlying buffer columns from being evicted, effectively creating a memory leak that will exhaust the Grid capacity and eventually crash the generator. Always use a `try...finally` block to guarantee `close()` is called.
-   **Concurrent Modification:** Do not share an NBufferBundle instance across multiple threads. The internal state is not protected by locks, and concurrent reads/writes will corrupt the data structures.
-   **Holding Access Objects:** Do not hold onto an Access object for longer than necessary. They are intended for immediate, scoped work. Holding them across different logical stages of generation prevents other stages from working on overlapping areas and inhibits memory reclamation.
-   **Reusing a Closed Access:** Attempting to use an Access object after its `close()` method has been called will result in assertion errors.

## Data Pipeline
The NBufferBundle does not form a data pipeline itself; rather, it serves as the central, stateful data repository that services a larger world generation pipeline.

> **Operational Flow:**
>
> Generator Stage (e.g., Terrain Carver) -> Requests `createBufferAccess(bounds)` -> **NBufferBundle** -> **Grid** (Checks cache) -> Creates/Loads necessary `NBuffer` columns -> Returns **Access** object -> Generator Stage modifies buffers via Access -> Generator Stage calls `Access.close()` -> Memory is unlocked for potential eviction.

