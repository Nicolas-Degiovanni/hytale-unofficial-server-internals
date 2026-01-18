---
description: Architectural reference for NoiseField
---

# NoiseField

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.noise
**Type:** Base Class / Contract

## Definition
```java
// Signature
public abstract class NoiseField {
```

## Architecture & Concepts
The NoiseField class is the abstract foundation for all procedural noise algorithms within the Hytale world generation system. It establishes a formal contract for querying a multi-dimensional, pseudo-random value at any given coordinate. This component is central to creating natural-looking, non-repeating environmental features such as terrain elevation, biome distribution, and cave formations.

Its primary architectural role is to decouple the specific noise algorithm (e.g., Perlin, Simplex, Worley) from the generator that consumes the noise data. Generators operate against the NoiseField contract, allowing world designers to swap out noise implementations without altering the higher-level generation logic.

The internal `scale` properties are a critical concept. They function as a coordinate space transformation layer. Before the core noise function is evaluated, input coordinates are multiplied by the corresponding scale values. This allows a single noise algorithm to be stretched, compressed, or tiled across different axes, providing powerful control over the frequency and characteristics of generated features.

## Lifecycle & Ownership
- **Creation:** Instances of this abstract class are never created directly. Concrete subclasses (e.g., PerlinNoiseField, SimplexNoiseField) are instantiated by higher-level systems, typically a WorldGenerator or BiomePlanner, often based on declarative world configuration files (e.g., JSON definitions).
- **Scope:** The lifetime of a NoiseField instance is typically bound to a specific, stateful generation task. For example, an instance may be created to generate a single region's heightmap and is discarded upon completion. It is not a long-lived, session-scoped service.
- **Destruction:** Ownership is managed by the creating generator. Once the generator completes its task and is garbage collected, all associated NoiseField instances are also reclaimed by the JVM Garbage Collector. There is no manual destruction or `close` method.

## Internal State & Concurrency
- **State:** NoiseField is a stateful class. It holds mutable `double` values for `scaleX`, `scaleY`, `scaleZ`, and `scaleW`. This state directly influences the output of all `valueAt` calls. The design intention is for the object to be configured once and then used for a series of related calculations.

- **Thread Safety:** This class is **not thread-safe**. The `setScale` methods modify internal state without any synchronization. If one thread calls `setScale` while another thread is calling `valueAt`, the results are non-deterministic and may be corrupted.

    **WARNING:** A single NoiseField instance should only be configured by one thread. Once configured, it can be safely passed to multiple worker threads for read-only `valueAt` operations. Any subsequent reconfiguration must be synchronized across all threads.

## API Surface
The public API provides methods for configuration and value retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| valueAt(double...) | double | O(1) | **Abstract.** Returns the noise value for a 1D, 2D, 3D, or 4D coordinate. Complexity is dependent on subclass implementation but is constant time per call. |
| setScale(double) | NoiseField | O(1) | Sets a uniform scale for all axes. Returns `this` for fluent configuration. |
| setScale(double...) | NoiseField | O(1) | Sets a unique scale for each axis. Returns `this` for fluent configuration. |

## Integration Patterns

### Standard Usage
The standard pattern involves instantiating a concrete subclass, configuring its scale fluently, and then using it within a generation loop to populate a data structure like a heightmap.

```java
// A generator obtains a concrete implementation
NoiseField terrainNoise = new PerlinNoiseField(seed);

// Configure the noise properties
terrainNoise.setScale(0.005); // Set a uniform scale for low-frequency terrain

// Use within a generation loop
for (int x = 0; x < CHUNK_SIZE; x++) {
    for (int z = 0; z < CHUNK_SIZE; z++) {
        double worldX = chunkOriginX + x;
        double worldZ = chunkOriginZ + z;

        // Query the noise value at a world coordinate
        double rawHeight = terrainNoise.valueAt(worldX, worldZ);

        // Map the noise value to a block height
        heightmap[x][z] = (int) (64 + rawHeight * 32);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call `setScale` from one thread while other threads might be calling `valueAt` on the same instance. This is a severe race condition.
- **Direct Instantiation of Base Class:** Do not attempt `new NoiseField()`. This will result in a compilation error as the class is abstract.
- **Stateful Reuse without Reset:** Avoid reusing a single NoiseField instance for generating unrelated features (e.g., terrain height and ore distribution) without explicitly re-configuring the scale. This can create unintended visual correlations in the world.

## Data Pipeline
NoiseField is a foundational data producer in the world generation pipeline. It transforms spatial coordinates into scalar noise values.

> Flow:
> World Coordinates (x, y, z) -> **NoiseField.valueAt()** -> Raw Noise Value (-1.0 to 1.0) -> Post-Processing (e.g., BiomeMapper, TerrainShaper) -> Final Block Data

