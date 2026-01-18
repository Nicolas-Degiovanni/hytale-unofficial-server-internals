---
description: Architectural reference for VectorWarpDensity
---

# VectorWarpDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class VectorWarpDensity extends Density {
```

## Architecture & Concepts
VectorWarpDensity is a fundamental operator within the procedural world generation engine. It functions as a "domain warping" node within a larger, directed acyclic graph (DAG) of Density objects. Its purpose is not to generate density values itself, but to controllably distort the input coordinates for another Density node.

This is a cornerstone technique for creating complex and natural-looking terrain. By displacing the sampling coordinates, it can stretch, twist, compress, or otherwise perturb the output of another function, such as a Perlin noise generator. This transforms simple, uniform patterns into the organic and chaotic shapes required for features like overhangs, stratified rock, and complex cave systems.

Architecturally, it implements a form of function composition. If the primary density function is *f(p)* and the warp function is *w(p)*, this node computes a new function *f'(p)* defined as:

*f'(p) = f(p + V * w(p) * F)*

Where:
- *p* is the original sample position (from Density.Context).
- *V* is the configured warpVector (direction of warp).
- *w(p)* is the result of processing the warpInput density (magnitude of warp).
- *F* is the scalar warpFactor.

This node is a pure data transformer; it receives a context, modifies it based on its inputs, and passes a new context to its child node.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level world generator service during the parsing and construction of a density graph. These graphs are typically defined in external configuration files (e.g., JSON assets) that describe the entire generation pipeline for a world or biome. It is never instantiated directly by gameplay code.
- **Scope:** The object's lifetime is bound to the density graph it is a part of. It is created once when the world generator is initialized and persists as long as that generator configuration is active.
- **Destruction:** Marked for garbage collection when the parent world generator is destroyed. This typically occurs on server shutdown or when loading a new world with a different generator configuration.

## Internal State & Concurrency
- **State:** The node's configuration (input, warpInput, warpFactor, warpVector) is stateful. This state is established at creation or via the setInputs method and is considered **effectively immutable** during the world generation process. The process method itself is stateless; it retains no information between invocations.
- **Thread Safety:** This class is **thread-safe and re-entrant**. The process method operates exclusively on its immutable configuration and the method-local arguments. Crucially, it creates a *new* Density.Context object for its child call, which prevents side effects and ensures that concurrent calls from different world generation threads do not interfere with each other. This design is critical for the massively parallel generation of world chunks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(C<sub>input</sub> + C<sub>warp</sub>) | Calculates the density of the input node at a displaced coordinate. The complexity is the sum of its own logic plus the recursive complexity of its input and warpInput nodes. |
| setInputs(Density[] inputs) | void | O(1) | Wires the input and warpInput nodes. This is used by the graph builder and is not intended for general use. Throws exceptions on invalid array length. |

## Integration Patterns

### Standard Usage
VectorWarpDensity is always used as an intermediate node in a density graph. It takes a primary density function (e.g., a noise generator) as its main input and a second density function to control the distortion.

```java
// Concept: Using a simple noise function to warp a more detailed one.

// 1. Define the base shape (e.g., layers of stone)
Density baseShape = new LayeredNoiseDensity(...);

// 2. Define the distortion field (e.g., large, slow noise)
Density distortionField = new PerlinNoiseDensity(low_frequency, ...);

// 3. Create the warp node
// Warp along the Y-axis to create vertical stretching/squashing
Vector3d verticalWarp = new Vector3d(0, 1, 0);
double warpStrength = 25.0;

Density warpedShape = new VectorWarpDensity(
    baseShape,          // The function to distort
    distortionField,    // The function controlling the distortion
    warpStrength,       // How much to distort
    verticalWarp        // The direction of distortion
);

// 4. The generator now processes warpedShape to get the final density
double finalDensity = warpedShape.process(worldGenContext);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never construct a VectorWarpDensity directly in game logic. Nodes must be created and wired by the world generator framework to ensure the integrity and performance of the density graph.
- **Stateful Inputs:** Do not provide Density inputs that are not themselves thread-safe. The thread-safety guarantee of VectorWarpDensity depends on the thread-safety of its inputs. A stateful, non-re-entrant input will break concurrency.
- **Cyclic Dependencies:** The density graph must be a DAG. Configuring a VectorWarpDensity to use itself as an input, either directly or indirectly, will cause a StackOverflowError during the process call.

## Data Pipeline
The flow of data for a single evaluation is a transformation of spatial coordinates.

> Flow:
> Initial Density.Context -> **VectorWarpDensity** `[Evaluates warpInput]` -> Warp Magnitude (double) -> `[Calculates new samplePoint]` -> New, Displaced Density.Context -> `[Passes to input.process()]` -> Final Density (double)

