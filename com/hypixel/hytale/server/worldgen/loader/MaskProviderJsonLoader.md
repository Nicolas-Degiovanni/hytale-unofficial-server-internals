---
description: Architectural reference for MaskProviderJsonLoader
---

# MaskProviderJsonLoader

**Package:** com.hypixel.hytale.server.worldgen.loader
**Type:** Transient

## Definition
```java
// Signature
public class MaskProviderJsonLoader extends JsonLoader<SeedStringResource, MaskProvider> {
```

## Architecture & Concepts

The MaskProviderJsonLoader is a specialized factory component within the server-side world generation pipeline. Its primary responsibility is to translate a declarative configuration, consisting of a JSON object and a source image file, into a functional, in-memory MaskProvider object.

This class acts as an orchestrator, bridging the gap between static worldgen assets and the runtime procedural engine. It does not contain the core masking logic itself. Instead, it composes and configures several other specialized components to build the final product:

*   **CoordinateRandomizerJsonLoader:** A sub-loader used to create a seeded randomizer for positional adjustments. This ensures that mask lookups can be deterministically jittered.
*   **PixelProvider:** A thin wrapper around the raw `BufferedImage` data, providing a clean interface for accessing pixel values.
*   **FuzzyZoom:** The core processing component that combines the image data, scaling parameters, and coordinate randomization to perform scaled, non-integer lookups against the source mask.

The resulting MaskProvider is consumed by higher-level world generation stages, such as biome placement or feature distribution, to influence where and how content is generated based on the grayscale values of the source image.

### Lifecycle & Ownership

*   **Creation:** An instance is created by a higher-level configuration parser or world generator orchestrator whenever it encounters a resource definition of this type. It is not a managed service or singleton.
*   **Scope:** The object's lifetime is exceptionally short. It is designed to be single-use, existing only for the duration of a single `load` operation.
*   **Destruction:** Once the `load` method returns the constructed MaskProvider, the MaskProviderJsonLoader instance has no further purpose and becomes eligible for standard garbage collection. No explicit cleanup is required.

## Internal State & Concurrency

*   **State:** This class is effectively **immutable**. Its internal fields are final and are populated exclusively by the constructor. It performs no internal caching and holds no mutable state between method calls. Each invocation of `load` executes a full, independent loading and composition pipeline from disk.

*   **Thread Safety:** The class is **conditionally thread-safe**. While an individual instance is not designed for concurrent use, its immutable nature means that multiple threads can safely create and use *separate instances* of MaskProviderJsonLoader simultaneously, provided the underlying file system supports concurrent reads.

    **Warning:** The `load` method performs blocking file I/O. It must **never** be called from a performance-critical thread, such as the main server tick loop. Its use is intended for the server's initialization or world generation phases.

## API Surface

The public contract is minimal, centered on the `load` method which executes the primary logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load() | MaskProvider | I/O-Bound | Constructs and returns a fully configured MaskProvider. This is the primary entry point. Throws a fatal `Error` if image loading or processing fails, indicating a critical world generation fault. |
| loadImage(Path) | static BufferedImage | I/O-Bound | A public static utility for loading an image from disk. Throws an `IOException` on failure. |

## Integration Patterns

### Standard Usage

This loader is intended to be instantiated and used immediately by a world generation system that parses configuration files. The caller is responsible for providing all necessary context, including paths and scaling parameters derived from the configuration.

```java
// Hypothetical usage within a worldgen orchestrator

// Parameters parsed from a worldgen JSON file
JsonElement maskConfig = ...;
Path assetFolder = ...;
Path imageFile = assetFolder.resolve("masks/continentalness.png");
Vector2i targetZoom = new Vector2i(4096, 4096);
Vector2i worldOffset = Vector2i.ZERO;
SeedString worldSeed = ...;

// Create, use, and discard the loader
MaskProviderJsonLoader loader = new MaskProviderJsonLoader(
    worldSeed,
    assetFolder,
    maskConfig,
    imageFile,
    targetZoom,
    worldOffset
);

MaskProvider continentalnessMask = loader.load();

// The 'continentalnessMask' is now ready for use in biome placement.
// The 'loader' instance is no longer needed.
```

### Anti-Patterns (Do NOT do this)

*   **Instance Caching:** Do not cache or reuse instances of MaskProviderJsonLoader. They are lightweight, transient objects designed for a single operation. Caching provides no performance benefit and complicates state management.

*   **Error Swallowing:** The `load` method throws a Java `Error` upon failure. This is intentional and signals a catastrophic, unrecoverable problem with the world generation assets. Do not wrap calls in a `try-catch(Throwable t)` block to suppress this. The error should be allowed to propagate up and halt the server startup, alerting the operator to a misconfiguration.

*   **Dynamic Loading:** Do not invoke this loader from the main game loop or in response to player actions. The blocking I/O will cause severe server stalls. All mask loading should occur during the initial, synchronous world generation phase.

## Data Pipeline

The class orchestrates a data transformation pipeline, converting static assets into a runtime object.

> Flow:
> (JsonElement, SeedString) -> CoordinateRandomizerJsonLoader -> CoordinateRandomizer
>
> Path -> **MaskProviderJsonLoader.loadImage** -> BufferedImage
>
> BufferedImage -> PixelProvider
>
> (CoordinateRandomizer, PixelProvider, zoom, offset) -> **MaskProviderJsonLoader.loadFuzzyZoom** -> FuzzyZoom
>
> FuzzyZoom -> **MaskProviderJsonLoader.load** -> MaskProvider
>
> MaskProvider -> World Generator Engine

