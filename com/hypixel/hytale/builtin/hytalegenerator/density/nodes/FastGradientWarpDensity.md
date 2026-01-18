---
description: Architectural reference for FastGradientWarpDensity
---

# FastGradientWarpDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Component/Node

## Definition
```java
// Signature
public class FastGradientWarpDensity extends Density {
```

## Architecture & Concepts
The FastGradientWarpDensity class is a fundamental component within Hytale's procedural world generation system. It functions as a **Decorator Node** in a directed acyclic graph (DAG) of density functions. Its primary role is not to generate density values itself, but to modify the spatial input for subsequent nodes in the graph.

This technique is known as **Domain Warping**. Instead of calculating a value at a given position *(x, y, z)*, this class first displaces the position to a new coordinate *(x', y', z')* and then queries its child node, the *input*, at that new location. This creates organic, flowing, and distorted features in the final terrain, such as swirling patterns or warped strata, which are difficult to achieve with simple noise combinations.

The displacement field is generated internally by a configured instance of the FastNoiseLite library, specifically using its gradient-based warping functions. This ensures that the distortion is smooth and continuous, avoiding the sharp artifacts that can result from simpler displacement methods.

## Lifecycle & Ownership
- **Creation:** Instances are typically created by a higher-level configuration loader or a world generator service. This class is not intended for direct, ad-hoc instantiation during the game loop. It is part of the static definition of a world generation pipeline.
- **Scope:** The object's lifetime is tied to the world generator configuration it is a part of. It persists as long as that generator is active for a given world or dimension.
- **Destruction:** The object is eligible for garbage collection when the generator graph it belongs to is discarded, for example, upon server shutdown or when a new world generation configuration is loaded. It has no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** FastGradientWarpDensity is a stateful object. Its behavior is defined by the immutable configuration provided at construction (warp scale, octaves, seed, etc.) and a mutable reference to its child *input* node. The internal FastNoiseLite instance holds a significant amount of configuration state.
- **Thread Safety:** This class is conditionally thread-safe and designed for parallel execution by the world generation engine. The core *process* method is re-entrant. It creates new, method-local context objects for its child calls, preventing state contamination between threads. The internal FastNoiseLite instance is only read from during processing, not written to. Therefore, a single configured FastGradientWarpDensity instance can be safely used by multiple worker threads simultaneously to process different world coordinates.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FastGradientWarpDensity(...) | constructor | O(1) | Configures and initializes the internal noise generator. Throws IllegalArgumentException for negative octaves. |
| process(Context) | double | O(octaves + C_input) | The primary execution method. Warps the input coordinates and invokes the child node's process method. |
| setInputs(Density[]) | void | O(1) | Connects this node to its child in the density graph. Part of the graph-building contract. |

## Integration Patterns

### Standard Usage
This class is used to wrap another Density node, forming a chain. The world generator builds this graph during its initialization phase.

```java
// Assume 'baseTerrain' is another Density node (e.g., PerlinNoiseDensity)
Density baseTerrain = new PerlinNoiseDensity(...);

// Wrap the base terrain with a warp effect
FastGradientWarpDensity warpedTerrain = new FastGradientWarpDensity(
    baseTerrain,
    2.0f,  // lacunarity
    0.5f,  // persistence
    3,     // octaves
    0.02f, // scale
    50.0f, // factor
    worldSeed
);

// In the generator, 'warpedTerrain.process(context)' is called, not 'baseTerrain.process'.
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Input Array:** The `setInputs` method is fragile. Providing an array with a length other than 1 will cause the internal *input* to become null, leading to the `process` method always returning 0.0. This can silently break the generation pipeline.
- **Processing Before Connection:** Calling `process` before a valid child node has been supplied via the constructor or `setInputs` will result in a default return value of 0.0, masking a critical configuration error.
- **State Mutation After Creation:** The internal FastNoiseLite instance is configured only once at construction. Attempting to modify the warp behavior after instantiation is not supported and requires creating a new FastGradientWarpDensity object.

## Data Pipeline
The flow of data through this component is a transformation of spatial coordinates.

> Flow:
> Density.Context (Original Position) -> **FastGradientWarpDensity** -> FastNoiseLite.DomainWarpFractalProgressive -> Density.Context (Warped Position) -> Child Density Node -> double (Density Value)

