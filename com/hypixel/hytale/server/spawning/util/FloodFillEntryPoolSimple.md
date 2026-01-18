---
description: Architectural reference for FloodFillEntryPoolSimple
---

# FloodFillEntryPoolSimple

**Package:** com.hypixel.hytale.server.spawning.util
**Type:** Utility

## Definition
```java
// Signature
public class FloodFillEntryPoolSimple {
```

## Architecture & Concepts
The FloodFillEntryPoolSimple class is a performance-critical memory management primitive. It implements a simple Object Pool pattern tailored specifically for integer arrays of a fixed size, defined by the ENTRY_SIZE constant.

Its primary function is to mitigate memory allocation overhead and reduce Garbage Collector (GC) pressure during high-frequency, computationally intensive algorithms, such as the server-side flood fill used for mob spawning validation or procedural generation tasks.

Instead of instantiating a new integer array for each node or step in an algorithm, a consumer can **allocate** a pre-existing array from the pool. Once the array's data is no longer needed, it is **deallocated**, returning it to the pool for immediate reuse. This pattern trades a small, persistent memory footprint for a significant reduction in GC churn and improved computational throughput.

## Lifecycle & Ownership
- **Creation:** Instantiated directly and locally by a higher-level system that performs a bounded, iterative task. For example, a single SpawnCalculator or WorldGeneratorTask would create its own private instance of this pool.
- **Scope:** The lifetime of the pool is strictly tied to the lifetime of the owning algorithmic task. It is designed to be short-lived, existing only for the duration of a single, complex calculation.
- **Destruction:** The pool object, along with all arrays it currently holds, becomes eligible for garbage collection as soon as the owning task completes and its reference is dropped. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** Highly mutable. The internal state consists of a list of available integer arrays. This list is continuously modified by allocate and deallocate operations. The size of the pool fluctuates based on the peak demand of the algorithm it serves.
- **Thread Safety:** **This class is not thread-safe.** It is built on a non-concurrent collection, ObjectArrayList, and lacks any synchronization mechanisms. It is fundamentally designed for use within a single, isolated thread. Concurrent access will lead to race conditions and unpredictable behavior.

**WARNING:** Sharing an instance of this class across multiple threads is a severe programming error that will result in data corruption and system instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| allocate() | int[] | O(1) | Retrieves a recycled array from the pool. If the pool is empty, a new array is allocated on the heap. |
| deallocate(int[] entry) | void | O(1) | Returns a previously allocated array to the pool, making it available for reuse. |

## Integration Patterns

### Standard Usage
This component is intended to be used as a local resource within a single method or class instance to manage memory for a specific algorithm.

```java
// Example: Within a single-threaded flood fill algorithm
FloodFillEntryPoolSimple pool = new FloodFillEntryPoolSimple();
Queue<int[]> workQueue = new ArrayDeque<>();

// To add a new coordinate to the queue
int[] entry = pool.allocate();
entry[0] = x;
entry[1] = y;
// ... populate remaining data ...
workQueue.add(entry);

// After processing an item from the queue
int[] processedEntry = workQueue.poll();
// ... use data from processedEntry ...
pool.deallocate(processedEntry); // CRITICAL: Return the array to the pool
```

### Anti-Patterns (Do NOT do this)
- **Shared Instance:** Do not register this class as a singleton or share a single instance across multiple systems or threads. Each task requiring a pool must create its own.
- **Leaking Entries:** Failure to call deallocate on every allocated array constitutes a memory leak from the pool's perspective. This negates the performance benefits, as the pool will be forced to create new arrays instead of recycling old ones.
- **Double Deallocation:** Calling deallocate more than once on the same array instance can lead to the same array being allocated to multiple consumers, causing severe data corruption.

## Data Pipeline
This class does not process data but rather manages the lifecycle of data containers. The flow is circular.

> Flow:
> Owning Algorithm requires a container -> **FloodFillEntryPoolSimple.allocate()** -> Array is populated and used by Algorithm -> Algorithm completes work with the array -> **FloodFillEntryPoolSimple.deallocate()** -> Array is returned to the internal pool for recycling.

