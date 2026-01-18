---
description: Architectural reference for Noise2dDensity
---

# Noise2dDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component

## Definition
```java
// Signature
public class Noise2dDensity extends Density {
```

## Architecture & Concepts
The Noise2dDensity class is a fundamental component within the procedural world generation system. It functions as a *source node* within a larger density graph, which is a network of operations used to calculate the final density value for any point in the world. A positive density typically corresponds to solid terrain, while a negative value corresponds to air.

This class acts as an adapter, bridging a raw NoiseField function (like Perlin or Simplex noise) to the standardized Density contract. Its primary architectural role is to introduce a two-dimensional noise pattern into the generation pipeline. By sampling the noise field using only the X and Z world coordinates, it produces values that are constant along the vertical Y-axis. This is a critical technique for generating large-scale, vertically-consistent features such as biome distributions, continent shapes, or foundational terrain elevations.

Because it ignores other density inputs, it is considered a *generator* or *leaf node* in the density graph tree.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly by the world generation configuration loader when processing a world generation preset. It is not managed by a dependency injection container and is constructed with its required NoiseField dependency.
-   **Scope:** The object's lifetime is bound to the world generator instance. It is created once during the generator's initialization phase and persists in memory until the entire generator is discarded, for example, on a server shutdown or when a different world is loaded.
-   **Destruction:** Cleanup is managed by the Java garbage collector. There are no native resources or explicit destruction methods. The object is eligible for collection once the root density graph is no longer referenced.

## Internal State & Concurrency
-   **State:** The internal state consists of a single reference to a NoiseField object. This reference is provided during construction and is immutable for the lifetime of the Noise2dDensity instance. The class itself is therefore effectively stateless post-initialization.
-   **Thread Safety:** This class is inherently thread-safe. The process method is a pure function relative to the class's own state. Its output is determined solely by the input Context and the behavior of the injected NoiseField. Concurrency safety is therefore delegated to the underlying NoiseField implementation. All standard engine-provided NoiseField implementations are guaranteed to be thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(1) | Samples the configured 2D noise field at the context's X and Z coordinates. The complexity is dependent on the underlying NoiseField but is typically constant time. |
| setInputs(Density[] inputs) | void | O(1) | No-op. This method is intentionally empty, as this node does not process inputs from other density nodes. Calling it has no effect. |

## Integration Patterns

### Standard Usage
A Noise2dDensity node is never used in isolation. It is constructed to wrap a configured noise function and then integrated into a larger, composite density graph where its output is combined with or modified by other nodes.

```java
// 1. Configure the underlying noise function for biome placement
NoiseField biomeNoise = new PerlinNoiseField(worldSeed, 0.005, 4);

// 2. Adapt the noise function for use in the density graph
Density biomeDensitySource = new Noise2dDensity(biomeNoise);

// 3. Use the node as an input to another operation (hypothetical)
// Density finalTerrain = new AddDensity(baseLandmass, biomeDensitySource);
// double finalValue = finalTerrain.process(worldPositionContext);
```

### Anti-Patterns (Do NOT do this)
-   **State Mutation:** Do not attempt to modify the internal NoiseField via reflection after construction. The immutability of the density graph is critical for predictable and reproducible world generation.
-   **Incorrect Dimensionality:** Do not use this node for features that require vertical variation, such as 3D caves, overhangs, or floating islands. Using Noise2dDensity for such features will result in undesirable, sheared, or column-like structures. Use a 3D-capable node like Noise3dDensity instead.

## Data Pipeline
Noise2dDensity acts as a starting point in a data flow. It transforms a world coordinate into a scalar density value that is then passed downstream to other nodes in the graph.

> Flow:
> Density Graph Traversal -> **Noise2dDensity.process(Context)** -> [Internal call to NoiseField.valueAt(x, z)] -> double (raw noise value) -> Upstream Combining Node (e.g., AddDensity, MultiplyDensity)

