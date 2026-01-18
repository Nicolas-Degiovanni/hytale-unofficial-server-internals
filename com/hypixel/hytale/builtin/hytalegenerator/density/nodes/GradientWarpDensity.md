---
description: Architectural reference for GradientWarpDensity
---

# GradientWarpDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Node / Component

## Definition
```java
// Signature
public class GradientWarpDensity extends Density {
```

## Architecture & Concepts
The GradientWarpDensity class is a fundamental component within the procedural world generation engine, specifically operating as a node within a Density Graph. Its primary function is not to generate density values itself, but to apply a sophisticated spatial distortion, or "domain warping," to an incoming density field.

This node implements a powerful procedural generation technique where the coordinate space itself is manipulated before sampling a density function. It operates on two distinct inputs:
1.  **Primary Input:** The main density field that will be sampled (e.g., a Perlin noise function).
2.  **Warp Input:** A secondary density field whose gradient (rate of change) is used to calculate a displacement vector.

At any given point in space, GradientWarpDensity calculates the gradient of the Warp Input by sampling it at several nearby locations. This gradient vector indicates the direction of the steepest ascent in the warp field. The vector is then scaled by a warp factor and used to offset the original coordinate. Finally, the node samples the Primary Input at this new, displaced coordinate.

The result is that the features of the Primary Input are pushed, pulled, and swirled according to the structure of the Warp Input, transforming simple patterns into complex, organic, and often turbulent-looking formations. This is essential for creating naturalistic features like warped stone strata, complex cave systems, and flowing terrain.

## Lifecycle & Ownership
-   **Creation:** An instance of GradientWarpDensity is created directly via its constructor during the construction of a density graph. This is typically handled by a higher-level builder or a generator configuration loader, which wires together various Density nodes to define a complete generation pipeline. It is not managed by a dependency injection container or service registry.
-   **Scope:** The object's lifetime is bound to the density graph it is a part of. It persists as long as the graph is required for a world generation task and is discarded afterward.
-   **Destruction:** Cleanup is managed by the Java garbage collector. When the root of the density graph is no longer referenced, all constituent nodes, including this one, become eligible for collection. There are no unmanaged resources requiring explicit destruction.

## Internal State & Concurrency
-   **State:** This class is stateful. It maintains references to its `input` and `warpInput` nodes, along with the configuration parameters `slopeRange` and `warpFactor`. This state is established during construction and can be altered post-construction via the `setInputs` method.

-   **Thread Safety:** **WARNING:** This class is not inherently thread-safe if its state is mutated. The `process` method is re-entrant and safe for concurrent execution *only under the assumption that the density graph structure is immutable during the generation phase*. The method internally creates new `Vector3d` and `Density.Context` objects for its calculations, preventing side effects on shared input data.

    However, calling `setInputs` from one thread while another thread is executing `process` on the same instance will lead to undefined behavior and likely a `NullPointerException` or inconsistent world generation. The expected operational pattern is to build the entire density graph once, and then process it in a read-only fashion across multiple worker threads.

## API Surface
The public contract is focused on data processing and initial configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(k) | Calculates the density at a warped coordinate. Complexity is dependent on the complexity of its input nodes, specifically requiring four calls to the warp input and one call to the primary input. |
| setInputs(Density[] inputs) | void | O(1) | Reconfigures the node's input connections. **WARNING:** Not safe to call during concurrent processing. |

## Integration Patterns

### Standard Usage
GradientWarpDensity is used to chain two density functions together, where one distorts the sampling space of the other. This is a core pattern for adding detail and realism to procedural noise.

```java
// Assume 'baseNoise' and 'warpNoise' are other initialized Density nodes.
// 'baseNoise' could define the general shape of rock or dirt.
// 'warpNoise' can be a lower-frequency noise to create large-scale swirling.

double slopeRange = 8.0;
double warpFactor = 32.0;

// Create the node, wiring the warp function to distort the base function.
Density warpedTerrain = new GradientWarpDensity(baseNoise, warpNoise, slopeRange, warpFactor);

// When the generator runs, it will call process on the final node.
// The context contains the world position to be evaluated.
double finalDensity = warpedTerrain.process(context);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call `setInputs` on a node that is part of a density graph currently being processed by worker threads. This will break thread safety guarantees and corrupt generation results.
-   **Recursive Connection:** Do not use the same `Density` instance for both the `input` and `warpInput` fields. While technically possible, this creates a recursive feedback loop that can lead to extreme, unpredictable, and computationally expensive results or stack overflows.
-   **Zero or Negative Slope Range:** The constructor throws an `IllegalArgumentException` if `slopeRange` is not positive. A small `slopeRange` captures fine-grained gradient details, while a large one captures broader changes. A non-positive value is mathematically invalid for this algorithm.

## Data Pipeline
The flow of data through this component for a single `process` call is a multi-stage coordinate transformation.

> Flow:
> 1.  `Density.Context` (with original `position`) enters `process`.
> 2.  The `warpInput` node is sampled at the original `position` and at three offset positions along the X, Y, and Z axes.
> 3.  The four samples are used to numerically approximate a gradient vector.
> 4.  The gradient vector is scaled by `warpFactor`.
> 5.  The scaled gradient is added to the original `position`, creating a **new warped position**.
> 6.  A new `Density.Context` is created containing this warped position.
> 7.  The new context is passed to the `input` node's `process` method.
> 8.  The final density value from the `input` node is returned.

