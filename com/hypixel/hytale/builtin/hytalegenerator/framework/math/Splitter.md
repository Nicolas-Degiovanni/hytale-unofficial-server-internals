---
description: Architectural reference for Splitter
---

# Splitter

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class Splitter {
```

## Architecture & Concepts

The Splitter class is a core mathematical utility within the world generation framework. Its sole purpose is to perform **spatial partitioning** on one-dimensional and two-dimensional integer-based domains. It provides a deterministic, stateless mechanism for subdividing large geometric primitives, specifically Range and Area, into smaller, contiguous chunks.

This capability is fundamental for parallelizing computationally expensive tasks, such as biome generation, terrain heightmap calculation, and structure placement. By dividing a large world area into a set of smaller, independent work units, the engine can distribute these units across multiple worker threads, significantly reducing the time required to generate a world.

The primary `split` method for an Area contains specialized logic to create more uniform, "squarish" sub-regions when the number of pieces is divisible by 2 or 3. This is a deliberate optimization to avoid generating long, thin "strip" regions, which can cause artifacts or inefficiencies in certain generation algorithms that operate on neighboring data.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, Splitter is never instantiated. The Java ClassLoader loads and initializes the class definition upon first access.
-   **Scope:** The class and its static methods are available for the entire lifetime of the application once loaded. It is globally accessible and has no instance-specific scope.
-   **Destruction:** The class definition is unloaded by the ClassLoader during application shutdown. There is no manual cleanup or destruction required.

## Internal State & Concurrency

-   **State:** The Splitter class is **stateless and immutable**. It contains no member fields and all its methods are pure functions; their output depends exclusively on their input arguments. The nested data structures it produces, Range and Area, are also immutable due to their `final` fields.

-   **Thread Safety:** This class is **inherently thread-safe**. Its stateless, pure-function design guarantees that it can be called concurrently from any number of threads without requiring locks, synchronization, or any other concurrency control mechanisms. This high-performance characteristic is critical for its role in a parallelized world generation pipeline.

## API Surface

The public contract consists of static factory methods for partitioning and the immutable data structures they operate on.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| split(Range range, int pieces) | Splitter.Range[] | O(N) | Partitions a 1D range into N pieces. Throws IllegalArgumentException if pieces is negative. |
| split(Area area, int pieces) | Splitter.Area[] | O(N) | Partitions a 2D area into N pieces, attempting to create a grid. Throws IllegalArgumentException if pieces is less than 1. |
| splitX(Area area, int pieces) | Splitter.Area[] | O(N) | Partitions a 2D area into N vertical strips along the X-axis. Throws IllegalArgumentException if pieces is less than 1. |
| new Area(minX, minZ, maxX, maxZ) | Splitter.Area | O(1) | Constructs an immutable 2D area. Throws IllegalArgumentException if max is less than min. |
| new Range(min, max) | Splitter.Range | O(1) | Constructs an immutable 1D range. Throws IllegalArgumentException if max is less than min. |

## Integration Patterns

### Standard Usage

The primary use case for Splitter is to prepare work for a parallel task scheduler or thread pool during world generation. A large area is defined and then broken down into a predictable number of chunks.

```java
// Define the total world area to be generated
Splitter.Area totalGenerationArea = new Splitter.Area(0, 0, 4096, 4096);

// Determine the number of available processor cores for parallel work
int workerCount = Runtime.getRuntime().availableProcessors();

// Divide the total area into a number of chunks matching the worker count
Splitter.Area[] workChunks = Splitter.split(totalGenerationArea, workerCount);

// Each 'chunk' can now be submitted to a separate worker thread for generation
// for (Splitter.Area chunk : workChunks) {
//    worldGenExecutor.submit(new GenerationTask(chunk));
// }
```

### Anti-Patterns (Do NOT do this)

-   **Excessive Partitioning:** Do not call `split` with a number of pieces significantly larger than the size of the area's dimensions. This will produce numerous empty or single-unit areas, adding unnecessary overhead. The implementation contains guards against this, but callers should be mindful of providing sensible inputs.
-   **Ignoring Return Value:** The methods in Splitter are pure functions with no side effects. Calling a split method without assigning its result to a variable is a logical error and accomplishes nothing.
-   **Re-implementing Logic:** Do not attempt to manually re-implement this partitioning logic elsewhere. This class provides a centralized, tested, and deterministic way to divide regions, ensuring all parts of the engine agree on how work is subdivided.

## Data Pipeline

Splitter acts as an initial "Fan-Out" stage in a data processing pipeline, specifically for world generation. It takes a single, large unit of work and transforms it into multiple, smaller units of work that can be processed in parallel.

> Flow:
> World Generation Request (defines total Area) -> **Splitter.split()** -> Array of smaller Area chunks -> Task Scheduler -> Parallel Biome/Terrain Generators

