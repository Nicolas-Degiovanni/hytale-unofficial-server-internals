---
description: Architectural reference for HeightThresholdInterpolator
---

# HeightThresholdInterpolator

**Package:** com.hypixel.hytale.server.worldgen.chunk
**Type:** Transient

## Definition
```java
// Signature
public class HeightThresholdInterpolator {
```

## Architecture & Concepts

The HeightThresholdInterpolator is a specialized, stateful worker responsible for smoothing terrain and biome data across a localized region during world generation. Its primary function is to eliminate the hard seams that would otherwise appear at chunk boundaries by sampling data from a larger area than the chunk being generated.

It operates as a "smoothing kernel" for the world generator. Before a chunk's block data is finalized, an instance of this class is created. It pre-fetches core world generation data (biome type, base height noise) for the target chunk *and* a surrounding border region (defined by MAX_RADIUS). This pre-fetched data is stored in a local grid cache.

The interpolator then processes this cached grid, calculating a weighted blend of biome influences for each point within the actual chunk boundaries. This blending process ensures that terrain features, such as hills and valleys, transition naturally from one biome to another. The final output is a smoothly interpolated height value, which the ChunkGenerator uses to determine the placement of solid blocks versus air.

This component is critical for procedural generation quality, acting as the bridge between raw noise functions and the final, aesthetically pleasing world geometry.

## Lifecycle & Ownership

-   **Creation:** An instance is created by the world generation pipeline, likely within a method responsible for generating a single chunk. It is always instantiated with a ChunkGeneratorExecution context, which anchors it to a specific location in the world.
-   **Scope:** The object's lifetime is extremely short and is strictly scoped to the generation of one chunk. It holds a significant memory footprint via its internal `entries` array and is not designed to be reused or persisted.
-   **Destruction:** The HeightThresholdInterpolator is eligible for garbage collection as soon as the chunk generation method that created it completes. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its primary internal state is the `entries` array, a two-dimensional grid of CoreDataCacheEntry objects. The `populate` method is designed specifically to mutate this array, filling it with raw and then interpolated world data. The object is useless until this state is fully computed.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for use by a single world generation worker thread. The internal `entries` array is accessed and modified without any synchronization mechanisms. Concurrent calls to `populate` or other methods will result in race conditions, data corruption, and unpredictable world generation artifacts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populate(int seed) | HeightThresholdInterpolator | O(N^2) | The primary initialization method. Fills the internal data grid by sampling and interpolating data. Must be called before any other methods. |
| getHeightNoise(int cx, int cz) | double | O(1) | Retrieves the final, interpolated height noise value for a local coordinate from the internal cache. |
| getHeightThreshold(int seed, int x, int z, int y) | float | O(1) | Calculates the final density threshold at a specific world coordinate. This is the primary output used for block placement. |
| getLowestNonOne(int cx, int cz) | int | O(1) | Retrieves a cached, biome-dependent vertical boundary value. |
| getHighestNonZero(int cx, int cz) | int | O(1) | Retrieves a cached, biome-dependent vertical boundary value. |

## Integration Patterns

### Standard Usage

The HeightThresholdInterpolator follows a strict "create, populate, use, discard" pattern. It is a temporary calculator, not a long-lived service.

```java
// 1. Obtain the execution context for the target chunk
ChunkGeneratorExecution execution = ...;
int seed = ...;

// 2. Create a new interpolator for this specific task
HeightThresholdInterpolator interpolator = new HeightThresholdInterpolator(execution);

// 3. Populate its internal cache. This is a mandatory step.
interpolator.populate(seed);

// 4. Use the populated data to generate the chunk
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        for (int y = 0; y < 256; y++) {
            // Get the final value to determine if a block should be solid
            float threshold = interpolator.getHeightThreshold(seed, globalX(x), globalZ(z), y);
            // ... logic to place block based on threshold
        }
    }
}

// 5. The interpolator is now out of scope and will be garbage collected.
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not attempt to re-use a HeightThresholdInterpolator instance for a different chunk. Its internal state is irrevocably tied to the coordinates provided by the initial ChunkGeneratorExecution.
-   **Premature Access:** Calling `getHeightThreshold` or any other data-retrieval method before `populate` has been successfully executed will result in undefined behavior, likely a NullPointerException or the use of uninitialized data.
-   **Concurrent Modification:** Never share an instance across multiple threads. The lack of internal locking will lead to severe data corruption in a multi-threaded context.
-   **Direct State Manipulation:** Avoid directly modifying the array returned by `getEntries`. This method exists for read-only access by tightly-coupled systems, and external modification breaks the class's invariants.

## Data Pipeline

The interpolator consumes raw, discrete data points from the ChunkGenerator and transforms them into a smooth, continuous field of values for a small area.

> Flow:
> ChunkGenerator -> Raw CoreDataCacheEntry -> **HeightThresholdInterpolator.populate()** -> Internal `entries` Cache (Blended Biome Data) -> **getHeightThreshold()** -> Final `float` Threshold -> Block Placement Logic

