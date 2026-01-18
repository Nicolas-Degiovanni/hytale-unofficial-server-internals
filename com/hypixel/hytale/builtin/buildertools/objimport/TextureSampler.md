---
description: Architectural reference for TextureSampler
---

# TextureSampler

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Utility

## Definition
```java
// Signature
public final class TextureSampler {
```

## Architecture & Concepts
The TextureSampler is a static utility class designed to abstract and optimize texture-related operations within the builder tools, specifically for processes like OBJ model importing. It serves two primary functions:

1.  **Texture Loading and Caching:** It provides a centralized mechanism to load image files from disk into memory. To prevent redundant and costly disk I/O, it maintains a global, in-memory static cache. Subsequent requests for the same texture path are served directly from this cache, significantly improving performance during large import operations.

2.  **Pixel Data Sampling:** It offers high-level methods to extract color and alpha information from a loaded image using normalized UV coordinates. This is a fundamental requirement for mapping a 2D texture onto a 3D model's surface. The class handles the necessary coordinate transformations, including vertical flipping of the V coordinate, which is a common convention.

This class acts as a low-level service for any system that needs to interpret texture data without directly managing file handles or image buffers.

## Lifecycle & Ownership
-   **Creation:** As a utility class with a private constructor, TextureSampler is never instantiated. Its static state, the `textureCache`, is initialized by the JVM ClassLoader when the class is first referenced.
-   **Scope:** The internal `textureCache` is static and therefore global. It persists for the entire lifetime of the application unless explicitly cleared.
-   **Destruction:** The cache is only cleared by an explicit call to the `clearCache` method.

**WARNING:** Failure to call `clearCache` after a large, one-time asset import can result in a significant memory leak, as all loaded `BufferedImage` objects will be retained in memory indefinitely.

## Internal State & Concurrency
-   **State:** The core state is a single static `Map<Path, BufferedImage>` named `textureCache`. This map is mutable and grows dynamically as new textures are loaded via the `loadTexture` method.

-   **Thread Safety:** **This class is not thread-safe.** The internal `textureCache` is a standard `HashMap`, which does not support concurrent modifications. The `loadTexture` method contains a non-atomic check-then-put sequence, creating a critical race condition. If two threads request the same uncached texture simultaneously, the texture may be loaded from disk twice, and the cache may be updated incorrectly.

**WARNING:** All access to this class from a multi-threaded context **must** be protected by external synchronization (e.g., a `synchronized` block or a `Lock`). Failure to do so will lead to unpredictable behavior and potential data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadTexture(Path) | BufferedImage | I/O Bound | Loads an image from disk. Returns a cached instance if available, otherwise loads, caches, and returns it. Returns null on failure. |
| sampleAt(BufferedImage, float, float) | int[] | O(1) | Samples the RGB color at the given UV coordinates. Returns an array of {R, G, B}. |
| sampleAlphaAt(BufferedImage, float, float) | int | O(1) | Samples the alpha value at the given UV coordinates. Returns 255 if the image has no alpha channel. |
| clearCache() | void | O(N) | Removes all entries from the internal texture cache, releasing their memory. N is the number of cached items. |
| getAverageColor(Path) | int[] | O(W*H) | Calculates the average RGB color of an image by iterating over all pixels. **This method bypasses the cache.** |

## Integration Patterns

### Standard Usage
The typical workflow involves loading a texture once, sampling it multiple times, and clearing the cache when the high-level operation is complete.

```java
// How a developer should normally use this
Path texturePath = Path.of("path/to/model/texture.png");

// Load the texture (uses cache)
BufferedImage texture = TextureSampler.loadTexture(texturePath);

if (texture != null) {
    // Sample the texture at various UV coordinates
    int[] color1 = TextureSampler.sampleAt(texture, 0.25f, 0.5f);
    int[] color2 = TextureSampler.sampleAt(texture, 0.8f, 0.1f);
    // ... perform more sampling
}

// After the import process is finished, clear the cache to free memory
TextureSampler.clearCache();
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Access:** Calling any method, especially `loadTexture`, from multiple threads without an external lock. This will cause race conditions.
-   **Repetitive Average Color Calculation:** Calling `getAverageColor` in a loop. This method is computationally expensive and performs disk I/O on every call, as it does not interact with the cache.
-   **Cache Negligence:** Forgetting to call `clearCache` after a large, self-contained process (like importing a scene) is complete. This will cause all loaded textures to remain in memory, leading to a memory leak.

## Data Pipeline
The flow of data for the primary sampling operation is linear, moving from a file path to a final color value.

> Flow:
> File Path -> `loadTexture` -> (Disk Read OR Cache Hit) -> `BufferedImage` -> `sampleAt` -> `int[] {R, G, B}`

