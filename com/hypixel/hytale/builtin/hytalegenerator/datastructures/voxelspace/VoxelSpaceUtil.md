---
description: Architectural reference for VoxelSpaceUtil
---

# VoxelSpaceUtil

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures.voxelspace
**Type:** Utility

## Definition
```java
// Signature
public class VoxelSpaceUtil {
```

## Architecture & Concepts
VoxelSpaceUtil is a stateless, high-performance utility class designed for orchestrating bulk data operations between VoxelSpace instances. It resides within the Hytale world generation pipeline and serves a critical role in optimizing the transfer of large volumetric datasets.

The primary architectural purpose of this class is to abstract and manage the complexity of parallel computation. By providing a simple, static interface for operations like copying, it decouples the core world generation logic from the underlying execution strategy. This allows generator stages to manipulate large regions of voxel data without needing to implement their own threading models, task scheduling, or error handling.

It leverages Java's CompletableFuture framework to partition the workload across multiple threads, maximizing CPU core utilization during computationally expensive generation phases. This is essential for maintaining acceptable world creation times.

## Lifecycle & Ownership
As a static utility class, VoxelSpaceUtil does not have a traditional object lifecycle.

- **Creation:** The class is loaded by the Java Virtual Machine's ClassLoader when first referenced. It is never instantiated.
- **Scope:** Its static methods are available globally throughout the application's lifetime once the class is loaded.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** VoxelSpaceUtil is entirely stateless. It contains no member fields and all its methods operate exclusively on the arguments provided at invocation time. All state is owned by the calling context and the VoxelSpace objects being manipulated.

- **Thread Safety:** The class itself is inherently thread-safe due to its stateless design. However, the operations it performs have significant concurrency implications for the data structures it manipulates. The `parallelCopy` method initiates concurrent write operations to the destination VoxelSpace. The internal partitioning logic ensures that each worker thread operates on a mathematically distinct set of voxel coordinates, preventing write conflicts at the utility level.

    **WARNING:** The underlying VoxelSpace implementation must be capable of handling concurrent writes to different coordinates. The caller is responsible for ensuring that no other threads are modifying the source or destination VoxelSpace objects while a copy operation is in progress.

## API Surface
The public API consists of a single static method for performing parallelized operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parallelCopy(VoxelSpace source, VoxelSpace destination, int concurrency) | void | O(N) | Executes a multi-threaded copy of all voxels from the source to the destination. N is the total number of voxels. Throws IllegalArgumentException if concurrency is less than 1. |

## Integration Patterns

### Standard Usage
This utility should be invoked during world generation stages where a large, immutable VoxelSpace needs to be transferred into a new, mutable VoxelSpace for further processing. The concurrency level should be tuned based on the target hardware profile, often tied to the number of available processor cores.

```java
// Standard invocation within a world generator stage
VoxelSpace<Block> template = getLandscapeTemplate();
VoxelSpace<Block> workingCopy = new ArrayVoxelSpace<>(template.getBounds());
int availableCores = Runtime.getRuntime().availableProcessors();

// Offload the expensive copy operation to the utility
VoxelSpaceUtil.parallelCopy(template, workingCopy, availableCores);

// 'workingCopy' is now ready for modification
carveCavesInto(workingCopy);
```

### Anti-Patterns (Do NOT do this)
- **Using Unsafe VoxelSpace Implementations:** Do not pass a destination VoxelSpace whose `set` method is not thread-safe. This will lead to race conditions, data corruption, or runtime exceptions.
- **External Modification:** Do not allow other threads to read from or write to either the source or destination VoxelSpace while a `parallelCopy` operation is executing. This utility provides no external locking mechanism; synchronization is the responsibility of the caller.
- **Excessive Concurrency:** Specifying a concurrency level significantly higher than the number of available CPU cores can degrade performance due to excessive context switching and thread management overhead.
- **Overlapping Source and Destination:** Passing the same VoxelSpace instance as both the source and destination will result in undefined behavior and is strictly unsupported.

## Data Pipeline
The `parallelCopy` method implements a data processing flow designed to maximize throughput by dividing the total voxel volume into discrete chunks and processing them in parallel.

> Flow:
> VoxelSpace (Source) -> **VoxelSpaceUtil.parallelCopy** -> Calculate Total Voxels -> Partition Workload by Concurrency -> Submit Tasks to ForkJoinPool -> Concurrent `source.getContent` / `destination.set` -> Join Futures -> Completion

