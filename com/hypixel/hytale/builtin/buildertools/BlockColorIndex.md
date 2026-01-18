---
description: Architectural reference for BlockColorIndex
---

# BlockColorIndex

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Transient

## Definition
```java
// Signature
public final class BlockColorIndex {
```

## Architecture & Concepts

The **BlockColorIndex** is a specialized, high-level utility service designed to facilitate programmatic world building, particularly for features that convert images or color data into in-game block structures. Its primary function is to provide an efficient lookup mechanism to find the closest matching in-game block for a given RGB color value.

Architecturally, this class acts as a derived and cached index built from the primary **BlockTypeAssetMap**. It does not store new information but rather transforms and organizes existing asset data into a query-optimized format for a specific use case: color matching.

The core design choice is the use of the CIELAB color space for all internal comparisons. Unlike the more common RGB color model, CIELAB is designed to be perceptually uniform. This means that the calculated geometric distance between two colors in the LAB space corresponds more accurately to human perception of color difference. This is critical for the system's primary goal of finding a block that *looks* closest to the target color, rather than one that is merely numerically close in RGB values.

The index is built on-demand using a lazy initialization pattern. The expensive process of iterating through all game assets, filtering for solid blocks, and performing color space conversions is deferred until the first time a lookup method is invoked.

## Lifecycle & Ownership

-   **Creation:** An instance is created via its public constructor, `new BlockColorIndex()`. It is expected to be instantiated by a higher-level manager or service responsible for builder tools.
-   **Scope:** The object is designed to be long-lived. The initial population of the index is a computationally expensive, one-time operation. To amortize this cost, a single instance should be created and retained for as long as its functionality is required, such as the duration of a user's building session or the lifetime of a server-side world generation task.
-   **Destruction:** The object holds no native resources and does not require explicit cleanup. It is garbage collected when all references to it are released.

## Internal State & Concurrency

-   **State:** The internal state of this class is mutable upon creation but becomes effectively immutable after its first use. The state consists of a list of **BlockColorEntry** records and an `initialized` boolean flag. The `ensureInitialized` method populates this list once and then prevents any further modification. This is a "write-once, read-many" state pattern.

-   **Thread Safety:** **WARNING:** This class is **NOT THREAD-SAFE**. The lazy initialization mechanism contains a critical race condition. If multiple threads call a public method on a non-initialized instance concurrently, the `ensureInitialized` block may be entered by all threads simultaneously. This will lead to unpredictable behavior, including data corruption in the internal list and excessive, redundant processing. Any system using this class in a multi-threaded context **MUST** provide its own external synchronization, either by pre-initializing the instance from a single thread or by wrapping all calls to its methods in a `synchronized` block.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findClosestBlock(r, g, b) | int | O(N) | The primary lookup method. Scans the entire index to find the block ID whose particle color is perceptually closest to the target RGB color. |
| findDarkerVariant(blockId, darkenAmount) | int | O(N) | Finds a block variant that is perceptually darker than the source block. Scans all blocks darker than the source to find the best match. |
| getBlockColor(blockId) | int | O(N) | Performs a reverse lookup to retrieve the packed integer RGB color for a given block ID. Returns -1 if not found. |
| findBlockForLerpedColor(...) | int | O(N) | Calculates a color by interpolating between two source colors in LAB space and finds the closest matching block to the result. |
| isEmpty() | boolean | O(1) | Checks if the index contains any valid block entries after initialization. |

*N = The total number of solid, cube-shaped blocks with a defined particle color in the game.*

## Integration Patterns

### Standard Usage

This class should be instantiated once by a parent service and reused for all subsequent color-based block lookups. The cost of the first call is high, while all subsequent calls are fast lookups against the in-memory cache.

```java
// A builder tool service would hold and manage the index instance
BlockColorIndex colorIndex = new BlockColorIndex();

// During an image-to-world conversion process...
for (Pixel pixel : sourceImage.getPixels()) {
    int r = pixel.getRed();
    int g = pixel.getGreen();
    int b = pixel.getBlue();

    // The first call to findClosestBlock will trigger the expensive initialization
    int blockId = colorIndex.findClosestBlock(r, g, b);

    world.setBlock(x, y, z, blockId);
}
```

### Anti-Patterns (Do NOT do this)

-   **Per-Operation Instantiation:** Do not create a new **BlockColorIndex** for every lookup. This defeats the purpose of the cache and will result in catastrophic performance degradation, as the entire block asset map will be scanned and processed on every call.

    ```java
    // ANTI-PATTERN: Extremely inefficient
    int blockId = new BlockColorIndex().findClosestBlock(r, g, b);
    ```

-   **Concurrent Initialization:** Do not share a single instance across multiple threads without external locking. As detailed in the Concurrency section, this will lead to race conditions and undefined behavior.

## Data Pipeline

The **BlockColorIndex** functions as a data transformation and caching layer. Its data flow is characterized by a one-time ingestion phase followed by a series of read-only query phases.

> **Initialization Flow (First call only):**
> **BlockTypeAssetMap** -> Iterate All Block Assets -> Filter (isSolidCube) -> Extract (ParticleColor) -> Convert (RGB to LAB) -> **BlockColorIndex Internal List** -> Sort by Lightness
>
> **Query Flow (Subsequent calls):**
> Input (RGB Color) -> Convert (RGB to LAB) -> **BlockColorIndex** -> Linear Scan & Compare (LAB Distance) -> Output (Closest Block ID)

