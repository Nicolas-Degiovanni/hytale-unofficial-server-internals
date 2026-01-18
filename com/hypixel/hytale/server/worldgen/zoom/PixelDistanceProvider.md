---
description: Architectural reference for PixelDistanceProvider
---

# PixelDistanceProvider

**Package:** com.hypixel.hytale.server.worldgen.zoom
**Type:** Stateful Utility

## Definition
```java
// Signature
public class PixelDistanceProvider {
```

## Architecture & Concepts
The **PixelDistanceProvider** is a spatial acceleration structure designed for extremely fast nearest-neighbor queries within a world generation context. Its primary function is to calculate the squared distance from any given coordinate to the nearest pixel of a *different* color within a source image, represented by a **PixelProvider**.

This class avoids costly full-image scans by pre-processing the source image into a grid of 8x8 pixel cells. During its construction, it analyzes each cell to determine the set of unique colors it contains. This pre-computed data is stored in an internal table.

The core optimization lies in how this table is used:
1.  When a query is made, the system only needs to check the cell containing the query point and its eight immediate neighbors (a 3x3 grid of cells).
2.  It uses the pre-computed color sets to instantly determine if a cell *could possibly* contain a relevant pixel. If a cell contains only colors identical to the query pixel's color, it is skipped entirely.
3.  Only when a neighboring cell is confirmed to contain different colors does the system perform a granular, 8x8 pixel scan *within that cell only*.

This two-level lookup (coarse grid check, then fine pixel check) makes query performance effectively constant time, regardless of the overall source image size. It is a critical component for performance-sensitive generation algorithms like biome edge blending or procedural feature placement based on distance fields.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by a higher-level world generation algorithm. It requires a fully populated **PixelProvider** during construction. The constructor performs the expensive, one-time analysis of the entire source image.
-   **Scope:** The object is transient and its lifetime is typically bound to a specific, self-contained generation stage. It should be created once per source **PixelProvider** and reused for all necessary distance queries against that provider.
-   **Destruction:** The object holds no external resources and is managed by the Java garbage collector. It becomes eligible for collection once the generator that created it completes its task and releases its reference.

## Internal State & Concurrency
-   **State:** The **PixelDistanceProvider** is highly stateful but becomes **effectively immutable** after its constructor completes. The internal `table` of pixel sets is computed once and never modified.

-   **Thread Safety:** This class is **thread-safe for all read operations**. Multiple threads can safely call **distanceSqToDifferentPixel** and **getColors** concurrently on a single instance without external synchronization.

    **WARNING:** The constructor is not thread-safe and performs significant work. The instance should be fully constructed by a single thread before being shared with other threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getColors() | IntSet | O(1) | Returns a set of all unique integer colors found in the source image during construction. |
| distanceSqToDifferentPixel(double ox, double oy, int px, int py) | double | O(1) | Calculates the squared distance from a coordinate (ox, oy) to the nearest pixel with a color different from the one at (px, py). Performance is independent of total image size. |

## Integration Patterns

### Standard Usage
The intended pattern is to create one **PixelDistanceProvider** for a given **PixelProvider** and reuse it for many queries. This amortizes the high initial cost of building the internal acceleration table.

```java
// Assume 'biomeMapProvider' is a fully generated PixelProvider
PixelDistanceProvider distanceProvider = new PixelDistanceProvider(biomeMapProvider);

// Later, in a feature placement algorithm...
for (int x = 0; x < width; x++) {
    for (int y = 0; y < height; y++) {
        // Find distance to the nearest different biome
        double distSq = distanceProvider.distanceSqToDifferentPixel(x + 0.5, y + 0.5, x, y);
        if (distSq < 25.0) {
            // This pixel is near a biome edge, apply blending...
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Re-creation in a Loop:** Do not create a new **PixelDistanceProvider** for each query. The constructor is expensive, and doing so negates all performance benefits.

    ```java
    // BAD: Incurs massive overhead
    for (int x = 0; x < width; x++) {
        PixelDistanceProvider provider = new PixelDistanceProvider(myImage); // DO NOT DO THIS
        double dist = provider.distanceSqToDifferentPixel(x, y, x, y);
    }
    ```

-   **Stale Provider Reference:** Do not modify the underlying **PixelProvider** after passing it to the **PixelDistanceProvider** constructor. The internal table will become stale, leading to incorrect calculations and subtle world generation bugs. The **PixelProvider** should be treated as immutable for the lifetime of the **PixelDistanceProvider**.

## Data Pipeline
The flow of data is a one-time ingestion followed by multiple fast queries.

> Flow:
> **PixelProvider** (Raw Image Data) -> Constructor Pre-processing -> **Internal Cell Table** (Acceleration Structure) -> **distanceSqToDifferentPixel** Query -> `double` (Result)

