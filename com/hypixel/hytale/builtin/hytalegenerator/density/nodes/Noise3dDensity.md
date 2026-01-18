---
description: Architectural reference for Noise3dDensity
---

# Noise3dDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class Noise3dDensity extends Density {
```

## Architecture & Concepts
The Noise3dDensity class is a foundational component within the procedural world generation system. It functions as a *source node* within a larger density graph, responsible for sampling a three-dimensional noise function at a given world coordinate.

Its primary role is to act as an adapter between a generic NoiseField object—which encapsulates a specific noise algorithm like Perlin or Simplex—and the abstract Density processing pipeline. By wrapping a NoiseField, it provides a standardized interface for the generator to query noise values, which are fundamental for creating natural-looking terrain features, caves, and other geological formations.

This class is a leaf node in the density graph; it generates a value from an external source (the NoiseField) and does not combine inputs from other Density nodes.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level generator configuration or a graph builder during the setup phase of world generation. It is always constructed with a pre-configured NoiseField instance, following a dependency injection pattern.
-   **Scope:** The lifetime of a Noise3dDensity instance is tied to the specific density graph it is part of. These graphs are typically constructed for a single, focused generation task (e.g., generating one world chunk) and are then discarded. It is not a long-lived or session-scoped object.
-   **Destruction:** The object is eligible for garbage collection as soon as the reference to its parent density graph is released. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal state consists of a single, final reference to a NoiseField object, which is set at construction time. The class itself does not maintain any mutable state between invocations of its process method.
-   **Thread Safety:** **Conditionally Thread-Safe**. The thread safety of this class is entirely dependent on the injected NoiseField implementation. If the provided NoiseField is thread-safe, which is standard for noise generation algorithms, then Noise3dDensity instances can be safely processed by multiple world generation threads concurrently. The class introduces no synchronization primitives of its own.

## API Surface
The public contract is minimal, focusing exclusively on the Density interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Samples the underlying NoiseField at the position specified in the context. Complexity is determined by the NoiseField implementation. |
| setInputs(Density[] inputs) | void | O(1) | No-op. This method is intentionally empty, as this node is a value generator and does not accept inputs from other density nodes. |

## Integration Patterns

### Standard Usage
This class is typically instantiated and composed into a larger graph by a factory or builder. The client code then invokes the process method on the graph's root node.

```java
// 1. Obtain a pre-configured noise field from a factory or registry
NoiseField terrainNoise = NoiseFieldFactory.createPerlinNoise(seed, frequency, octaves);

// 2. Construct the density node via dependency injection
Density noiseSource = new Noise3dDensity(terrainNoise);

// 3. The generator invokes process() at a specific world coordinate
Density.Context generationContext = new Density.Context(new Vector3d(450.5, 82.0, -1200.2));
double densityValue = noiseSource.process(generationContext);

// The returned value is then used by other systems
if (densityValue > 0.5) {
    // place stone
}
```

### Anti-Patterns (Do NOT do this)
-   **Stateful NoiseField:** Do not inject a NoiseField that is not thread-safe or that maintains mutable state. Doing so will produce non-deterministic and inconsistent terrain artifacts when the generator is run across multiple threads.
-   **Re-implementing Combiner Logic:** Do not attempt to modify this class to combine its output with other values. Its purpose is strictly to sample noise. Use dedicated combiner nodes like AddDensity or MultiplyDensity to build complex functions.

## Data Pipeline
Noise3dDensity serves as an entry point for data into the density graph. It transforms a spatial coordinate into a scalar density value.

> Flow:
> World Coordinate (from Density.Context) -> **Noise3dDensity**.process() -> NoiseField.valueAt() -> Raw Density Value (double) -> Upstream Combiner Node or Graph Output

