---
description: Architectural reference for PixelProvider
---

# PixelProvider

**Package:** com.hypixel.hytale.server.worldgen.zoom
**Type:** Transient Data Model

## Definition
```java
// Signature
public class PixelProvider {
```

## Architecture & Concepts

The PixelProvider is a fundamental data structure within the server-side world generation pipeline. It serves as a high-performance, in-memory representation of a 2D grid of integer data, abstracting the complexities of a raw one-dimensional array into a more intuitive 2D coordinate system.

Its primary role is to hold and provide mutable access to intermediate data during procedural generation steps, such as biome mapping, heightmap generation, or temperature calculations. Each integer in the grid typically represents a specific data value, like a biome ID or a terrain height.

Key architectural characteristics include:

*   **Bounds Clamping:** The `getPixel` method implements edge clamping. Requests for coordinates outside the defined width and height do not throw an exception; instead, they return the value of the nearest pixel on the edge. This is a critical feature for generation algorithms that use kernels or sample neighboring pixels, as it simplifies the logic at chunk boundaries.
*   **Column-Major Order:** The internal one-dimensional `pixels` array stores data in a column-major layout, calculated as `x * height + y`. This is a deliberate design choice that can impact cache performance depending on the iteration patterns of the generation algorithms that consume it.
*   **Alpha Channel Stripping:** The constructor that accepts a BufferedImage explicitly masks the incoming pixel data with `& 16777215` (0x00FFFFFF). This operation removes the alpha channel, ensuring that the provider only stores 24-bit RGB data. This implies the system is not designed to handle transparency in its core generation data.

## Lifecycle & Ownership

-   **Creation:** A PixelProvider is a short-lived object. It is typically instantiated at the beginning of a specific world generation stage (e.g., a `ZoomLayer`). Creation occurs either by sampling a source `BufferedImage` or by creating a deep copy of a `PixelProvider` from a previous stage using the `copy()` method.
-   **Scope:** Its lifetime is confined to the execution of a single, discrete generation algorithm or a small chain of related algorithms. It is passed between methods, mutated, and then typically discarded.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the generation stage that created it completes and no longer holds a reference. There are no manual resource management or `close()` methods.

## Internal State & Concurrency

-   **State:** The PixelProvider is **highly mutable**. While the `width` and `height` dimensions are final and established at construction, the core `pixels` array is designed for frequent modification via the `setPixel` method.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without external synchronization. It is designed for synchronous, single-threaded access within a generation task. Concurrent calls to `setPixel` or modification of the array returned by `getPixels` will result in race conditions and undefined behavior.

**WARNING:** The `getPixels()` method returns a direct reference to the internal array. Modifying this array externally bypasses all class logic and is extremely dangerous.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPixel(int x, int y) | int | O(1) | Retrieves the pixel value at the given coordinates. Clamps out-of-bounds coordinates to the nearest edge. |
| setPixel(int x, int y, int pixel) | void | O(1) | Mutates the pixel value at the given coordinates. Does not perform bounds checking and will throw an `ArrayIndexOutOfBoundsException` if coordinates are invalid. |
| copy() | PixelProvider | O(width * height) | Creates and returns a deep copy of the instance, including a full copy of the pixel data array. |
| getPixels() | int[] | O(1) | Returns a direct reference to the internal pixel data array. Use with extreme caution. |

## Integration Patterns

### Standard Usage

The typical pattern involves creating a PixelProvider, running a generation algorithm that reads from and writes to it, and then passing the result to the next stage, often by creating a new copy.

```java
// A generator receives a provider from a previous stage
PixelProvider inputProvider = worldGenContext.getPreviousStageOutput();

// Create a new provider for the output of this stage
PixelProvider outputProvider = new PixelProvider(inputProvider.getWidth(), inputProvider.getHeight());

// Process each pixel
for (int x = 0; x < inputProvider.getWidth(); x++) {
    for (int y = 0; y < inputProvider.getHeight(); y++) {
        int neighborValue = inputProvider.getPixel(x - 1, y); // Safely reads clamped edge value
        int newValue = calculateNewValue(neighborValue);
        outputProvider.setPixel(x, y, newValue);
    }
}

// The outputProvider is now ready for the next stage
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not access a single PixelProvider instance from multiple threads. The lack of internal locking will lead to data corruption. Each thread must operate on its own distinct instance.
-   **External Array Mutation:** Avoid modifying the array returned by `getPixels()`. This is a performance escape hatch, but it breaks encapsulation and is a common source of bugs. Always prefer using the `setPixel` method.
-   **Reference Mismanagement:** Do not mistake assignment for a copy. Assigning a PixelProvider to a new variable creates a reference, not a new object. Unintended modifications can occur if you do not use the explicit `copy()` method when a unique data set is required.

## Data Pipeline

The PixelProvider acts as a stateful data container that is transformed at each step of a larger generation pipeline.

> Flow:
> Source `BufferedImage` -> **PixelProvider (Initial State)** -> ZoomLayer 1 (Mutates State) -> **PixelProvider (State 2)** -> ZoomLayer 2 (Mutates State) -> **PixelProvider (Final State)** -> World Chunk Data

