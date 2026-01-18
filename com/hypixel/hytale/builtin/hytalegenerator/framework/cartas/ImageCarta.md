---
description: Architectural reference for ImageCarta
---

# ImageCarta

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.cartas
**Type:** Transient

## Definition
```java
// Signature
public class ImageCarta<R> extends TriCarta<R> {
```

## Architecture & Concepts
The ImageCarta is a specialized implementation of the TriCarta interface, designed to translate 2D image data into 3D world data. It acts as a bridge between artistic, image-based input (like a biome map or feature placement guide) and the procedural world generator.

Its core architectural function is to sample a source image at a coordinate determined by applying noise functions to the input world position (x, y, z). The resulting RGB color from the image is then used as a key to look up a specific world-generation value, typically a terrain type or biome identifier.

This design pattern allows for complex, non-linear mapping of world space onto the image, preventing simple tiling or grid-like artifacts. By using noise functions for the projection, it ensures that the features defined in the image appear organically within the 3D world. It is a fundamental component for data-driven and artist-directed procedural generation.

## Lifecycle & Ownership
- **Creation:** An ImageCarta instance is created exclusively through the nested ImageCarta.Builder. The constructor is private to enforce this pattern. The builder must be fully configured with a source image, two noise functions, and at least one RGB-to-value mapping before the build method is called.

- **Scope:** The object's lifetime is typically bound to a single, complete world generation task. It is instantiated during the generator's initialization phase, used extensively during chunk generation, and then becomes eligible for garbage collection once the generator's work is complete. It is not a long-lived, session-wide service.

- **Destruction:** Managed entirely by the Java garbage collector. The class holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state of an ImageCarta is **effectively immutable** after construction. The Builder pattern populates all fields, including the source image's pixel data into an internal integer array. No public methods exist to modify this state post-construction. This immutability is a critical design feature.

- **Thread Safety:** The class is **thread-safe for all read operations**. Due to its immutable state, the primary method, apply, can be called concurrently from multiple worker threads without any risk of race conditions or data corruption. No external locking or synchronization is required. This makes it highly suitable for use in parallelized world generation engines.

## API Surface
The public API is minimal, focusing on the core data retrieval function. Static color utility methods are provided for convenience but are not part of the instance contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(x, y, z, tHreadId) | R | O(1) | The primary function. Maps a 3D world coordinate to a configured value of type R by sampling the internal image. Returns null if the sampled color has no mapping. |
| allPossibleValues() | List<R> | O(1) | Returns a pre-calculated list of all possible non-null values that the apply method can return. |

## Integration Patterns

### Standard Usage
The ImageCarta must be constructed and configured using its builder. The resulting instance is then used by a system that queries 3D world coordinates.

```java
// 1. Obtain noise functions and a source image
TriDoubleFunction<Double> noiseX = ...;
TriDoubleFunction<Double> noiseY = ...;
BufferedImage biomeMapImage = loadImage("biomes.png");

// 2. Configure the builder with mappings
ImageCarta.Builder<Biome> builder = new ImageCarta.Builder<>();
builder.withImage(biomeMapImage);
builder.withNoiseFunctions(noiseX, noiseY);
builder.addTerrainRgb(ImageCarta.coloursToRgb(0, 255, 0), Biome.FOREST);
builder.addTerrainRgb(ImageCarta.coloursToRgb(255, 255, 0), Biome.DESERT);

// 3. Build the immutable ImageCarta instance
ImageCarta<Biome> biomeCarta = builder.build();

// 4. Use the carta within the world generator
Biome currentBiome = biomeCarta.apply(worldX, worldY, worldZ, threadId);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to instantiate with `new ImageCarta()` will result in a compile-time error. Always use the builder.
- **Incomplete Configuration:** Calling `build()` on the builder before providing an image via `withImage` or noise functions via `withNoiseFunctions` will throw an `IllegalStateException`.
- **Builder Reuse:** While technically possible, reusing a single builder instance to create multiple, differently configured ImageCarta objects is not recommended. It is safer and clearer to create a new builder for each distinct configuration.

## Data Pipeline
The data flow for a single `apply` call is a direct, multi-stage lookup process.

> Flow:
> 3D World Coordinate (x, y, z) -> Noise Functions -> 2D Image Coordinate (sampleX, sampleY) -> **ImageCarta** (rgbArray lookup) -> RGB Integer -> **ImageCarta** (rgbToTerrainMap lookup) -> Output Value (R)

