---
description: Architectural reference for SumNoiseProperty
---

# SumNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class SumNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The SumNoiseProperty class is a core component of the procedural generation library, implementing the Composite design pattern for noise functions. Its primary role is to combine multiple, distinct NoiseProperty instances into a single, more complex noise value. This is achieved by calculating a weighted sum of the outputs from each constituent noise function.

This layering capability is fundamental for creating sophisticated and natural-looking procedural content. For example, a low-frequency base noise for continental shapes can be combined with a medium-frequency noise for mountains and a high-frequency noise for surface details.

By conforming to the NoiseProperty interface, a SumNoiseProperty instance can be used interchangeably with any other NoiseProperty. This allows for the recursive composition of noise functions, enabling the construction of deeply nested and intricate generation logic from simple, reusable building blocks. The final output is normalized via the GeneralNoise.limit function, ensuring that the resulting value remains within the engine's expected bounds.

## Lifecycle & Ownership
- **Creation:** SumNoiseProperty objects are typically instantiated by configuration loaders or procedural assemblers during the initialization phase of a world generation task. They are often the result of deserializing data-driven definitions (e.g., from JSON or HOCON files) that describe the desired noise layers.
- **Scope:** The lifetime of a SumNoiseProperty instance is bound to the specific generation context for which it was created. It is not a global or session-level object. It persists as long as the generator configuration is needed, such as for the duration of a single world or biome generation process.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection once the generation task is complete and all references to it are released. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** The primary state is the `entries` array, which holds the collection of child NoiseProperty objects and their associated weighting factors. This array is marked as final and is assigned only once during construction.
- **Thread Safety:** **NOT THREAD-SAFE**. This class is unsafe for concurrent modification. While the `get` methods themselves are re-entrant, the nested Entry objects are mutable. If one thread calls a `get` method while another thread modifies an Entry's factor or NoiseProperty reference, the behavior is undefined.

    **WARNING:** Treat SumNoiseProperty instances as effectively immutable after construction. All configuration should be completed before the object is passed to a multi-threaded generation system. Failure to do so can result in non-deterministic generation, visual artifacts, and race conditions.

## API Surface
The public API is minimal, focusing exclusively on the NoiseProperty contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Calculates the 2D noise value by summing the weighted results of N child noise properties. |
| get(seed, x, y, z) | double | O(N) | Calculates the 3D noise value by summing the weighted results of N child noise properties. |
| getEntries() | Entry[] | O(1) | Returns the internal array of Entry objects. **Warning:** Exposes mutable internal state. |

## Integration Patterns

### Standard Usage
The intended use is to define a composite noise function by combining other NoiseProperty instances. This is typically done once during a setup phase.

```java
// Assume baseNoise and detailNoise are existing NoiseProperty objects
SumNoiseProperty.Entry baseLayer = new SumNoiseProperty.Entry(baseNoise, 0.75);
SumNoiseProperty.Entry detailLayer = new SumNoiseProperty.Entry(detailNoise, 0.25);

// Create the composite noise property
NoiseProperty combinedNoise = new SumNoiseProperty(new SumNoiseProperty.Entry[]{
    baseLayer,
    detailLayer
});

// Use the combined noise in a generator
double value = combinedNoise.get(worldSeed, 100.5, 25.2);
```

### Anti-Patterns (Do NOT do this)
- **Post-Construction Modification:** Modifying the state of a SumNoiseProperty after it has been constructed and shared is a severe anti-pattern. The internal Entry objects are mutable and exposed via the getEntries method. Modifying them during a generation process will lead to unpredictable results.

    ```java
    // DANGEROUS: Do not do this after the object is in use
    SumNoiseProperty noise = ...;
    noise.getEntries()[0].setFactor(0.5); // This will affect all subsequent calls
    ```

- **Concurrent Access and Modification:** Never modify an Entry from one thread while another thread might be calling a `get` method on the parent SumNoiseProperty. This is a classic race condition.

## Data Pipeline
SumNoiseProperty acts as a transformation and aggregation node within a larger procedural generation pipeline. It does not source data but rather processes inputs from other nodes.

> Flow:
> Generator Configuration -> Deserializer -> Creates **SumNoiseProperty** with child NoiseProperty instances -> World Generator requests value at (x,y,z) -> **SumNoiseProperty** queries children -> Aggregated Noise Value -> Voxel Generation Logic

