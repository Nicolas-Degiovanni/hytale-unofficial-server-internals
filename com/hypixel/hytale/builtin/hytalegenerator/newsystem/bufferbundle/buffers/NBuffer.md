---
description: Architectural reference for NBuffer
---

# NBuffer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class NBuffer implements MemInstrument {
```

## Architecture & Concepts
NBuffer is an abstract base class that establishes the fundamental contract for all specialized data buffers within the Hytale World Generator's *New System*. It is not a concrete, usable component on its own but rather the architectural blueprint from which tangible buffers, such as VoxelBuffer or BiomeDataBuffer, must inherit.

Its primary role is to enforce a unified interface for memory management and introspection across the entire world generation pipeline. By implementing the MemInstrument interface, every NBuffer subclass is guaranteed to be a trackable component within the engine's performance and memory profiling systems. This design is critical for managing the high-volume, transient data allocations that occur during procedural generation, ensuring that memory usage can be precisely monitored, controlled, and optimized.

NBuffer and its derivatives form the core data transport layer for a single generation task, acting as containers for raw, unstructured data as it moves between generation stages like noise application, biome assignment, and structure placement.

## Lifecycle & Ownership
The lifecycle described here applies to **concrete implementations** of NBuffer, not the abstract class itself.

-   **Creation:** Concrete NBuffer instances are exclusively created and managed by a centralized buffer pooling system, such as a BufferBundleManager. They are never instantiated directly. A generation worker thread requests a buffer of a specific type and size from the pool at the beginning of a task.
-   **Scope:** The lifetime of an NBuffer instance is intentionally brief, scoped strictly to a single, discrete generation job (e.g., the generation of one world chunk). It is considered "owned" by the worker thread that acquired it.
-   **Destruction:** True destruction is rare. Upon completion of a generation task, the buffer is not garbage collected but is instead "released" back to the central pool. This return-to-pool mechanism is paramount for minimizing memory allocation overhead and GC pressure, which are major performance bottlenecks in large-scale procedural generation.

## Internal State & Concurrency
-   **State:** Any concrete NBuffer is inherently **mutable**. Its purpose is to be an empty container that is progressively filled with data by generation algorithms. The internal state typically consists of a large, primitive array (e.g., byte[], int[]) or a direct-allocated ByteBuffer.
-   **Thread Safety:** NBuffer instances are **not thread-safe** and are designed for single-threaded access. The architectural pattern is one of thread-local ownership. A worker thread acquires a buffer, performs all write operations, and then hands off ownership to the next stage of the pipeline.

    **WARNING:** Sharing an NBuffer instance between multiple threads without external locking will result in race conditions and catastrophic data corruption. The system relies on a strict ownership transfer protocol.

## API Surface
The public API is defined by the MemInstrument interface, which all subclasses must implement.

| Symbol            | Type   | Complexity | Description                                                                                             |
| :---------------- | :----- | :--------- | :------------------------------------------------------------------------------------------------------ |
| getMemoryUsage()  | long   | O(1)       | Returns the total allocated memory size of the buffer in bytes. Critical for performance monitoring.      |
| getMemoryHandle() | Object | O(1)       | Provides an opaque handle to the underlying memory structure for advanced, low-level diagnostics.       |

## Integration Patterns

### Standard Usage
The correct pattern involves acquiring a buffer from a managing service, operating on it, and releasing it upon completion. This ensures proper pooling and memory tracking.

```java
// Acquire a specific buffer type from the central manager
VoxelBuffer voxelData = bufferManager.acquire(VoxelBuffer.class);

// A single worker thread populates the buffer
worldgenAlgorithm.populate(voxelData);

// Pass the buffer to the next system (e.g., meshing)
mesher.process(voxelData);

// IMPORTANT: Release the buffer back to the pool
bufferManager.release(voxelData);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ConcreteNBuffer()`. This bypasses the memory pooling system, leading to severe GC pressure, memory fragmentation, and a loss of performance tracking.
-   **Cross-Thread Sharing:** Do not access or modify the same NBuffer instance from multiple threads concurrently. The system is not designed for this and lacks internal synchronization.
-   **Forgetting to Release:** Failure to release a buffer back to its pool after use constitutes a memory leak. The buffer will be lost to the system for the duration of the session, degrading the pool's capacity and performance.

## Data Pipeline
NBuffer serves as the primary data container that flows between the stages of the world generation system.

> Flow:
> Generation Algorithm -> **Writes to Concrete NBuffer** -> Meshing System -> Reads from NBuffer -> Render Chunk Builder -> Consumes Meshing Output

