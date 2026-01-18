---
description: Architectural reference for NPixelBuffer
---

# NPixelBuffer

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.bufferbundle.buffers
**Type:** Abstract Data Structure

## Definition
```java
// Signature
public abstract class NPixelBuffer<T> extends NBuffer {
```

## Architecture & Concepts
NPixelBuffer is an abstract base class that defines a contract for a fixed-size, three-dimensional data grid. It serves as a fundamental data structure within the new world generation system, representing a small, discrete volume of space, specifically an 8x1x8 region.

This class is a cornerstone of the generator's data pipeline, acting as a standardized container for intermediate data passed between different generation stages. The generic type parameter *T* allows concrete implementations to store various types of per-coordinate data, such as biome identifiers, block states, or heightmap values.

By extending NBuffer, it participates in the BufferBundle system, which aggregates multiple NPixelBuffer instances (each for a different data type) into a single, cohesive unit of work for a specific world region. This design decouples generator stages, allowing one stage to write biome data into an NPixelBuffer without needing to know how a subsequent stage will read that data to carve terrain.

The static constants BUFFER_SIZE_BITS and SIZE enforce a uniform geometry for all pixel buffers, which is critical for predictable and performant coordinate calculations within the generator.

## Lifecycle & Ownership
- **Creation:** Instances of concrete NPixelBuffer subclasses are not created directly. They are instantiated and managed by a parent BufferBundle during the initialization of a world generation task for a specific chunk column.
- **Scope:** The lifetime of an NPixelBuffer is strictly tied to its parent BufferBundle. It is a transient object that exists only for the duration of a single, localized generation operation.
- **Destruction:** The object is eligible for garbage collection as soon as the BufferBundle that owns it is discarded. This typically occurs after all relevant generation stages have processed the data and it has been committed to a more permanent world structure. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is defined by the contract but managed by concrete implementations. The state is inherently **mutable**, consisting of the grid of *T* values. Implementations are expected to hold a data array or a similar structure to store the 64 (8x1x8) pixel content values.
- **Thread Safety:** **Not thread-safe.** An NPixelBuffer instance is designed to be operated on by a single world generation thread at a time. Concurrent reads or writes from multiple threads will lead to race conditions and undefined behavior. All synchronization must be handled externally at a higher level, typically by ensuring that a given BufferBundle is only processed by one worker thread.

## API Surface
The public API provides a simple, coordinate-based contract for accessing a 3D data grid.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPixelContent(Vector3i) | T | O(1) | Retrieves the data content at the specified local coordinates. Returns null if no content is set. |
| setPixelContent(Vector3i, T) | void | O(1) | Sets the data content for the specified local coordinates. Passing null is permitted to clear the content. |
| getPixelType() | Class<T> | O(1) | Returns the runtime class of the generic type T. Used for type-safe retrieval from non-generic contexts. |

## Integration Patterns

### Standard Usage
The canonical use case involves retrieving a specific buffer implementation from a BufferBundle within a generator stage and then reading or writing data to it.

```java
// A generator stage receives a bundle for the region it's processing
BufferBundle bundle = context.getBufferBundle();

// Request a specific type of pixel buffer (e.g., for biomes)
NBiomeBuffer biomeBuffer = bundle.getBuffer(NBiomeBuffer.class);

// Iterate over the local volume and set data
for (int x = 0; x < 8; x++) {
    for (int z = 0; z < 8; z++) {
        Vector3i localPos = new Vector3i(x, 0, z);
        Biome someBiome = calculateBiomeForPosition(localPos);
        biomeBuffer.setPixelContent(localPos, someBiome);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of an NPixelBuffer subclass using its constructor. The BufferBundle system is responsible for the lifecycle of these objects. Direct creation bypasses the framework and will lead to unmanaged, isolated data.
- **Cross-Thread Access:** Do not share an NPixelBuffer instance across multiple threads without external locking. The design assumes a single-threaded access pattern per generation task.
- **Out-of-Bounds Access:** The buffer represents a fixed 8x1x8 volume. Attempting to call getPixelContent or setPixelContent with coordinates outside this range will result in an error or undefined behavior in the underlying implementation. All coordinates must be local to the buffer.

## Data Pipeline
NPixelBuffer is a critical link in the chain of data flow between world generation stages. It acts as a temporary, in-memory database for a small region of the world.

> Flow:
> Biome Generation Stage -> **NBiomeBuffer.setPixelContent()** -> Terrain Height Stage -> **NBiomeBuffer.getPixelContent()** -> Cave Generation Stage -> ... -> Final Chunk Assembler

