---
description: Architectural reference for SpatialData
---

# SpatialData<T>

**Package:** com.hypixel.hytale.component.spatial
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class SpatialData<T> {
```

## Architecture & Concepts
SpatialData is a high-performance, generic container designed for storing and sorting collections of objects associated with 3D coordinates. It is a fundamental building block for any system that performs spatial queries, such as physics engines, culling systems, or AI.

The architecture is intentionally optimized for CPU cache efficiency by implementing a **Structure of Arrays (SoA)** pattern. Instead of a single array of objects each containing a vector and data (Array of Structures), SpatialData maintains separate, parallel arrays for vectors, user data, and sorting metadata. This layout ensures that when an algorithm iterates over positions (e.g., during sorting), it accesses a contiguous block of memory, minimizing cache misses.

A key architectural feature is the use of an `indexes` array as an **indirection layer**. When a sort operation is performed, the primary `vectors` and `data` arrays are not modified. Instead, the `indexes` array is reordered to reflect the sorted sequence. This avoids the significant performance cost of swapping large objects in memory and preserves the original insertion order if needed. Consumers of this class must iterate through the `indexes` array to access data in a sorted manner.

The class provides two sorting algorithms:
1.  A standard lexicographical sort on vector components (X, then Z, then Y).
2.  A **Morton Order** sort (also known as a Z-order curve). This technique maps 3D coordinates to a 1D value in a way that preserves spatial locality. Sorting by Morton codes is a powerful optimization for algorithms that partition space or require efficient nearest-neighbor searches, as it groups objects that are close in 3D space next to each other in the sorted array.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor (`new SpatialData<T>()`). It is not a managed service or singleton. Systems that process collections of entities, such as a rendering or collision detection system, will typically create an instance as needed.
- **Scope:** The lifecycle is determined entirely by its owner. It is most often used as a short-lived, temporary buffer. For example, an instance might be created at the beginning of a frame, populated with relevant data, processed, and then cleared or discarded at the end of the frame.
- **Destruction:** The object is eligible for garbage collection as soon as it falls out of scope. The `clear` method is provided to facilitate object reuse, which is a critical pattern for performance in game engines. Calling `clear` resets the logical size to zero but retains the allocated capacity of the internal arrays, preventing the overhead of repeated memory allocation and deallocation across frames.

## Internal State & Concurrency
- **State:** SpatialData is a highly **mutable** stateful container. Its primary purpose is to be populated and sorted, which inherently modifies its internal arrays and size counter. The backing arrays grow dynamically as needed but are never shrunk, even by the `clear` method, to optimize for reuse. The `sortMorton` method also populates a temporary `moroton` array as a cache for the calculated Morton codes.
- **Thread Safety:** This class is **not thread-safe**. All methods that modify state (`add`, `append`, `sort`, `clear`) do so without any synchronization. Concurrent access from multiple threads will result in race conditions, data corruption, and unpredictable exceptions. It must be confined to a single thread or protected by external locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(vector, value) | void | Amortized O(1) | Adds an element. Triggers internal array growth if capacity is exceeded, which is an O(N) operation. |
| addCapacity(size) | void | O(N) | Ensures internal arrays have enough capacity for at least N more elements. Use this to prevent reallocations in tight loops. |
| append(vector, value) | void | O(1) | Adds an element without checking capacity. **Warning:** Unsafe. Throws an exception if capacity is insufficient. |
| sort() | void | O(N log N) | Sorts the internal index map lexicographically by vector components (X, then Z, then Y). |
| sortMorton() | void | O(N log N) | Sorts the internal index map using Morton codes to achieve spatial locality. |
| clear() | void | O(N) | Resets the logical size to 0 and nulls out data references. The internal arrays retain their capacity for reuse. |
| getSortedIndex(i) | int | O(1) | Retrieves the true index from the sorted index map. This is the primary way to access sorted data. |
| getVector(i) | Vector3d | O(1) | Retrieves the vector at a given raw index. |
| getData(i) | T | O(1) | Retrieves the data at a given raw index. |

## Integration Patterns

### Standard Usage
The canonical use case is to populate, sort, process, and clear the data structure within a single scope, such as a system's update method. To access sorted data, you must iterate from 0 to `size()` and use `getSortedIndex` to look up the actual index in the data arrays.

```java
// A system pre-allocates a SpatialData instance to reduce garbage.
private final SpatialData<Entity> visibleEntities = new SpatialData<>();

public void processVisibleEntities(List<Entity> allEntities) {
    // 1. Populate the data structure
    for (Entity entity : allEntities) {
        if (entity.isVisible()) {
            visibleEntities.add(entity.getPosition(), entity);
        }
    }

    // 2. Sort for efficient processing (e.g., front-to-back rendering)
    visibleEntities.sort();

    // 3. Process the sorted data using the indirection layer
    for (int i = 0; i < visibleEntities.size(); i++) {
        int sortedIndex = visibleEntities.getSortedIndex(i);
        Vector3d pos = visibleEntities.getVector(sortedIndex);
        Entity entity = visibleEntities.getData(sortedIndex);
        // ... perform rendering or other logic ...
    }

    // 4. Clear for the next frame
    visibleEntities.clear();
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Index Map:** Do not iterate directly over the `vectors` or `data` arrays after calling a sort method. This will process the elements in their original insertion order, defeating the purpose of the sort. Always use `getSortedIndex`.
- **Unsafe Appending:** Do not call `append` inside a loop without first guaranteeing capacity with `addCapacity`. This is a fragile optimization that will lead to `ArrayIndexOutOfBoundsException` if the exact size is miscalculated. Prefer `add` unless performance is absolutely critical and the size is known.
- **Frequent Allocation:** Do not create a `new SpatialData()` instance every frame inside a main game loop. This generates significant garbage and pressure on the garbage collector. Instead, store it as a member field and reuse it by calling `clear()` each frame.
- **Concurrent Modification:** Do not share a SpatialData instance across threads without external synchronization. It is designed for single-threaded access.

## Data Pipeline
SpatialData is not a pipeline processor itself, but rather a critical data structure used as a stage within a larger data processing pipeline. It acts as a gathering and sorting point before data is consumed by a subsequent system.

> Flow:
> Collection of Game Entities -> **SpatialData (Population)** -> **SpatialData (Morton Sort)** -> Broad-Phase Physics System -> Collision Pair Generation

