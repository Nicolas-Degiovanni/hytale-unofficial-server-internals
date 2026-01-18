---
description: Architectural reference for NormalizerDensity
---

# NormalizerDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component / Transient

## Definition
```java
// Signature
public class NormalizerDensity extends Density {
```

## Architecture & Concepts
The NormalizerDensity class is a fundamental component within the procedural world generation's **Density Graph**. It functions as a transformation node, or filter, that remaps a numerical range. Its primary role is to take the output value from an upstream Density node and scale it to a new, specified range.

This class is not a source of density data itself; it is an adapter that modifies data flowing through the graph. For example, a raw Perlin noise function might output values between -1.0 and 1.0. A NormalizerDensity node can be used to translate this range into a practical world-height range, such as 60 to 180 blocks. This allows for the decoupling of raw noise generation from its application, enabling designers to tune world features without altering the underlying noise algorithms.

It is a core building block for composing complex terrain features, biome gradients, and resource distributions from simpler procedural functions.

### Lifecycle & Ownership
-   **Creation:** NormalizerDensity instances are constructed by the world generation framework during the assembly of a Density Graph. This typically occurs when the system parses a world generation preset or configuration file. It is not intended to be instantiated directly by high-level game logic.
-   **Scope:** The lifetime of an instance is tied to the lifetime of the Density Graph it belongs to. It persists as long as the graph is required for generating world data, typically for the duration of a single world generation session or for a specific region's generation task.
-   **Destruction:** Instances are marked for garbage collection when the parent Density Graph is dereferenced. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** Mutable. The normalization parameters (fromMin, fromMax, toMin, toMax) are immutable and set at construction time. However, the reference to the upstream **input** Density node is mutable and can be changed post-construction via the setInputs method. This allows for dynamic reconfiguration of the Density Graph, but introduces thread-safety concerns.

-   **Thread Safety:** **Not thread-safe for modification.** The class lacks internal synchronization. Modifying the graph by calling setInputs while another thread is executing the process method will lead to undefined behavior and potential race conditions.
    -   **WARNING:** The Density Graph containing this node must be treated as immutable during concurrent processing. All graph configuration should be completed in a single-threaded context before submitting it to a multi-threaded world generation engine. The process method itself is re-entrant and safe for concurrent execution, provided the graph structure remains static.

## API Surface
The public API is minimal, focusing exclusively on data processing and graph composition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(C) | Processes the input Density node and remaps its output value. Complexity is dependent on the complexity (C) of the entire upstream node chain. Throws NullPointerException if the input node is null. |
| setInputs(Density[] inputs) | void | O(1) | Sets the upstream data source. Only the first element of the array is used. Passing an empty array will set the input to null, causing subsequent calls to process to fail. |

## Integration Patterns

### Standard Usage
NormalizerDensity is used to wrap another Density node, transforming its output. It is almost always used as part of a larger chain of operations.

```java
// Assume 'perlinNoise' is a pre-existing Density node generating values from -1 to 1.
// We want to map this to a terrain height between 64 and 192.
Density perlinNoise = new PerlinNoise(...);

// Create the normalizer to wrap the noise function.
// The 'input' argument in the constructor is the standard way to link nodes.
Density terrainHeight = new NormalizerDensity(-1.0, 1.0, 64.0, 192.0, perlinNoise);

// When the world generator calls process, the remapped value is returned.
double height = terrainHeight.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
-   **Graph Modification During Processing:** Never call setInputs on a NormalizerDensity node that is part of a Density Graph currently being used by active generator threads. This will corrupt the data pipeline.
-   **Nullifying Input:** Setting the input to null via `setInputs(new Density[0])` is a valid operation but primes the node for failure. Any subsequent call to `process` will result in a `NullPointerException`. Inputs should only be set to valid Density instances.
-   **Invalid Range:** The constructor throws an IllegalArgumentException if a minimum value is greater than its corresponding maximum value. This validation should not be bypassed.

## Data Pipeline
The class acts as a simple, stateless transformation step in a larger data flow. It receives a single floating-point value and outputs a single transformed floating-point value.

> Flow:
> Upstream Density Node (e.g., PerlinNoise) → `process()` → Raw double value [-1.0, 1.0] → **NormalizerDensity** → Normalized double value [64.0, 192.0] → Downstream Consumer (e.g., TerrainVoxelPlacer)

