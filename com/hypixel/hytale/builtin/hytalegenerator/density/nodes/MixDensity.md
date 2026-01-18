---
description: Architectural reference for MixDensity
---

# MixDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Node

## Definition
```java
// Signature
public class MixDensity extends Density {
```

## Architecture & Concepts
The MixDensity class is a fundamental compositor node within the procedural world generation's density function system. It operates as a conditional interpolator, blending the outputs of two distinct density functions, referred to as A and B. The blending is controlled by a third function, the *influence density*.

This class embodies the "blending" or "transition" pattern crucial for creating natural-looking terrain. For instance, it can be used to smoothly transition from a plains biome (Density A) to a mountain biome (Density B) using a Perlin noise function as the influence map.

Architecturally, MixDensity is a node in a directed acyclic graph (DAG) of Density objects. The `process` method triggers a recursive evaluation of its subgraph, fetching values from its three inputs to compute a final, blended output. It is one of the primary tools for combining and layering generator features.

## Lifecycle & Ownership
- **Creation:** Instances are typically created by a higher-level configuration loader or a world generator service when it constructs the density graph from a blueprint. The three required child Density nodes are provided during construction.
- **Scope:** The lifetime of a MixDensity instance is bound to the specific world generator configuration it is part of. It is not a global or session-scoped object and persists only as long as its parent generator is in memory.
- **Destruction:** The object is reclaimed by the Java garbage collector when its parent world generator is unloaded or replaced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** MixDensity is stateful. Its state consists of the object references to its three child nodes: densityA, densityB, and influenceDensity. This state is mutable via the `setInputs` method, though this is intended for initial graph construction, not for runtime modification.
- **Thread Safety:** This class is **conditionally thread-safe**. The `process` method does not modify any internal fields, making the node itself safe for concurrent reads. However, thread safety is entirely dependent on the thread safety of the provided `Density.Context` object and the child Density nodes. The world generation engine is expected to parallelize chunk generation by providing each worker thread with a unique, thread-local `Context` object, which makes concurrent calls to a shared MixDensity instance safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | double | O(N) | Recursively evaluates the subgraph to compute a blended density value. N is the total number of nodes in its subgraph. |
| setInputs(Density[] inputs) | void | O(1) | Reconfigures the node with new inputs. Throws IllegalArgumentException if the input array length is not exactly 3. |

## Integration Patterns

### Standard Usage
The primary use case is to blend two terrain features using a third as a mask or controller.

```java
// Context: Within a world generator's initialization logic.
// Define the two densities to be blended.
Density baseTerrain = new FlatlandDensity(-5.0);
Density mountainTerrain = new MountainPeakDensity(120.0);

// Define the density that will control the blend.
// Output near 0.0 = baseTerrain, near 1.0 = mountainTerrain.
Density influenceMap = new PerlinNoiseDensity(seed, scale);

// Create the MixDensity node to perform the interpolation.
MixDensity finalTerrain = new MixDensity(baseTerrain, mountainTerrain, influenceMap);

// During world generation, this node is processed for each coordinate.
double densityValue = finalTerrain.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Cyclical Dependencies:** Constructing a density graph where a child node (e.g., densityA) eventually references its parent MixDensity will create an infinite recursion. This will cause a StackOverflowError when `process` is called. The graph must be a DAG.
- **Invalid Input Configuration:** Calling `setInputs` with an array whose length is not 3 will raise an unrecoverable IllegalArgumentException at runtime.
- **Runtime Re-Wiring:** While `setInputs` is public, modifying the density graph while it is being actively processed by multiple threads is not safe and will lead to unpredictable generation artifacts. Graph modification should only occur during initialization.

## Data Pipeline
MixDensity is a computational node, not a data-transfer node. Its pipeline is a flow of recursive function calls and floating-point calculations.

> Flow:
> World Generator Request -> **MixDensity.process(context)** -> [Recursive call to influenceDensity.process] -> [Conditional recursive calls to densityA.process and densityB.process] -> Linear Interpolation -> Final double value

