---
description: Architectural reference for DoubleRangeNoiseSupplier
---

# DoubleRangeNoiseSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Transient / Strategy

## Definition
```java
// Signature
public class DoubleRangeNoiseSupplier implements IDoubleCoordinateSupplier {
```

## Architecture & Concepts
The DoubleRangeNoiseSupplier is a fundamental component within the procedural generation engine. It serves as a compositional bridge, adapting a raw noise function to a specific, bounded output range. Its primary architectural role is to decouple the noise algorithm (e.g., Perlin, Simplex) from the application of that noise (e.g., determining terrain height, cave density, or temperature).

This class embodies the Strategy pattern. It is configured with two strategies at construction:
1.  **NoiseProperty:** The source of the raw, unbounded noise values.
2.  **IDoubleRange:** The function that maps the raw noise values into a predictable, usable range (e.g., mapping -1.0 to 1.0 into 0.0 to 255.0).

By combining these two dependencies, it provides a concrete implementation of the **IDoubleCoordinateSupplier** interface, which is consumed by higher-level world generation systems that require coordinate-based values.

### Lifecycle & Ownership
-   **Creation:** Instances are created and configured by higher-level procedural systems, such as a **BiomeGenerator** or **FeaturePlacementService**. It is not a singleton; many distinct instances will exist simultaneously, each configured for a specific purpose (e.g., one for elevation, another for humidity).
-   **Scope:** The lifetime of a DoubleRangeNoiseSupplier is tied to the lifetime of its owning configuration object. It persists as long as the procedural generation recipe that defines it is in scope.
-   **Destruction:** The object is managed by the garbage collector. It holds no native resources and requires no explicit destruction. It is reclaimed once the parent generator or biome definition is unloaded.

## Internal State & Concurrency
-   **State:** This class is effectively immutable. Its dependencies, **IDoubleRange** and **NoiseProperty**, are assigned in the constructor and are not modified thereafter. The class itself maintains no mutable state between calls.

-   **Thread Safety:** This class is conditionally thread-safe. It contains no internal locks or mutable state, making it safe for concurrent access **if and only if** its underlying **IDoubleRange** and **NoiseProperty** dependencies are also thread-safe. Given its use in world generation, these dependencies are expected to be safe for parallel execution by multiple chunk generation threads.

    **WARNING:** Providing a non-thread-safe **NoiseProperty** or **IDoubleRange** to this class will lead to race conditions and non-deterministic world generation.

## API Surface
The public contract is defined by the **IDoubleCoordinateSupplier** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(1) | Generates a 2D noise value, mapped to the configured range. Complexity is dependent on the underlying noise algorithm. |
| get(seed, x, y, z) | double | O(1) | Generates a 3D noise value, mapped to the configured range. Complexity is dependent on the underlying noise algorithm. |

## Integration Patterns

### Standard Usage
This class should be instantiated as part of a larger configuration and then used as an implementation of **IDoubleCoordinateSupplier**. It is typically held as a field within a generator class.

```java
// Within a Biome configuration or World Generator
// 1. Define the raw noise function
NoiseProperty elevationNoise = new PerlinNoiseProperty(...);

// 2. Define the mapping function (e.g., map to world heights 64-192)
IDoubleRange heightRange = new LinearDoubleRange(64.0, 192.0);

// 3. Compose them into the supplier
IDoubleCoordinateSupplier elevationSupplier = new DoubleRangeNoiseSupplier(heightRange, elevationNoise);

// 4. Use the supplier to get a final value for a coordinate
double worldHeight = elevationSupplier.get(worldSeed, 1024.5, 2048.1);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in a Loop:** Do not create new instances of DoubleRangeNoiseSupplier inside a tight loop (e.g., for each block in a chunk). It is a lightweight object, but repeated instantiation is inefficient. Create it once and reuse it.
-   **Stateful Dependencies:** Do not inject dependencies (**NoiseProperty**, **IDoubleRange**) that rely on mutable internal state. This will break thread safety and result in unpredictable generation artifacts.

## Data Pipeline
The flow of data for a single **get** call is linear and synchronous. The class orchestrates the transformation of coordinates into a final, range-mapped value.

> Flow:
> Input Coordinates & Seed -> **DoubleRangeNoiseSupplier.get()** -> NoiseProperty.get() -> Raw Noise Value [-1.0, 1.0] -> IDoubleRange.getValue() -> Final Mapped Value [min, max] -> Consumer (World Generator)

