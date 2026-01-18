---
description: Architectural reference for FillerBlockUtil, the core server utility for multi-block structure logic.
---

# FillerBlockUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class FillerBlockUtil {
```

## Architecture & Concepts

FillerBlockUtil is a stateless, static utility class that provides the fundamental logic for handling **multi-block structures**. In the Hytale engine, certain game objects like large doors, beds, or crafting stations occupy a volume larger than a single 1x1x1 block. The engine represents these objects with a single "base" block and several "filler" blocks that occupy the remaining volume. This class contains the algorithms to validate the integrity of these structures and iterate over their constituent block positions.

The core concept is the **filler integer**. This is a packed integer that encodes the relative 3D offset of a filler block from its base block. FillerBlockUtil uses a 15-bit packing scheme (5 bits per axis) to store this offset, allowing for relative coordinates from -16 to +15. This is a significant memory optimization, as it avoids storing full 3D vector objects for every filler block in a chunk.

The class is decoupled from any specific world data implementation (e.g., chunks, world snapshots) through the **FillerFetcher** interface. This is a Strategy Pattern that allows the caller to provide a data source-specific implementation for retrieving block data. This makes the validation logic universally applicable, whether checking a newly generated chunk, a player's building action, or a world edit operation.

## Lifecycle & Ownership

-   **Creation:** As a static utility class with a private constructor, FillerBlockUtil is never instantiated. The class is loaded by the JVM at runtime.
-   **Scope:** Its static methods are available globally throughout the server application's lifecycle.
-   **Destruction:** The class is unloaded when the server's JVM shuts down. There is no instance-level data to clean up.

## Internal State & Concurrency

-   **State:** FillerBlockUtil is completely **stateless**. It contains no mutable fields and all operations are performed on data provided as method arguments. All fields are static final constants.

-   **Thread Safety:** This class is **inherently thread-safe**. Its stateless nature ensures that concurrent calls from multiple threads (e.g., world generation workers, player interaction threads) will not interfere with each other.

    **WARNING:** While the utility itself is thread-safe, the thread safety of the overall operation depends entirely on the provided FillerFetcher implementation. If the fetcher reads from a data source that is not safe for concurrent access, the caller must implement their own synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachFillerBlock(boxes, consumer) | void | O(N) | Iterates every integer block coordinate within a bounding box. N is the volume of the box. |
| testFillerBlocks(boxes, predicate) | boolean | O(N) | Tests every integer block coordinate against a predicate. Short-circuits and returns false on the first failure. |
| validateBlock(..., fetcher) | ValidationResult | O(N) | The primary validation routine. Checks if a block is a valid base or filler part of a multi-block structure. |
| pack(x, y, z) | int | O(1) | Encodes a relative 3D coordinate into a single packed filler integer. |
| unpackX(val) | int | O(1) | Decodes the X component from a packed filler integer. |
| unpackY(val) | int | O(1) | Decodes the Y component from a packed filler integer. |
| unpackZ(val) | int | O(1) | Decodes the Z component from a packed filler integer. |

## Integration Patterns

### Standard Usage

The primary use case is validating a region of the world. A system like a ChunkManager would implement the FillerFetcher interface to read its own data and pass it to the validateBlock method.

```java
// Hypothetical validation of a block within a Chunk
Chunk targetChunk = world.getChunkAt(chunkPos);

// The FillerFetcher is implemented using lambdas to read from the chunk
FillerBlockUtil.FillerFetcher<Chunk, Void> fetcher = new FillerBlockUtil.FillerFetcher<>() {
    @Override
    public int getBlock(Chunk chunk, Void v, int x, int y, int z) {
        return chunk.getBlockIdAt(x, y, z);
    }

    @Override
    public int getFiller(Chunk chunk, Void v, int x, int y, int z) {
        return chunk.getFillerDataAt(x, y, z);
    }

    @Override
    public int getRotationIndex(Chunk chunk, Void v, int x, int y, int z) {
        return chunk.getRotationAt(x, y, z);
    }
};

// Perform the validation
ValidationResult result = FillerBlockUtil.validateBlock(
    worldX, worldY, worldZ,
    targetChunk.getBlockIdAt(worldX, worldY, worldZ),
    targetChunk.getRotationAt(worldX, worldY, worldZ),
    targetChunk.getFillerDataAt(worldX, worldY, worldZ),
    targetChunk, null, fetcher
);

if (result != ValidationResult.OK) {
    // Handle invalid multi-block structure
    log.warn("Detected invalid filler block at " + worldX);
}
```

### Anti-Patterns (Do NOT do this)

-   **Misinterpreting the Filler Integer:** The `filler` parameter in `validateBlock` expects the packed integer coordinate, not a block ID or other metadata. Using the wrong value will lead to validation failures. Always use the `pack` method or data retrieved from a valid world source.
-   **Ignoring Validation Results:** A result of INVALID_BLOCK or INVALID_FILLER indicates world data corruption or a bug in a block placement system. This must be handled by either correcting the data or logging a critical error. Ignoring it can lead to cascading failures in physics, rendering, and AI.
-   **Unsafe Fetcher Implementation:** Providing a FillerFetcher that reads from a non-thread-safe data structure in a multi-threaded context (like parallel chunk generation) will cause race conditions and unpredictable behavior. The fetcher implementation must be thread-safe if the context is concurrent.

## Data Pipeline

The flow of data during a standard validation operation is linear and synchronous. The utility acts as a pure function that transforms world state information into a validation status.

> Flow:
> World Data Source (e.g., Chunk) -> **FillerFetcher** (Adapter) -> **FillerBlockUtil.validateBlock** -> ValidationResult Enum -> Calling System (e.g., World Generator, Block Placement Service)

