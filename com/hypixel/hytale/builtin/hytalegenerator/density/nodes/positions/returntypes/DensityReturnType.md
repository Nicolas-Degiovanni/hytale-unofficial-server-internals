---
description: Architectural reference for DensityReturnType
---

# DensityReturnType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions.returntypes
**Type:** Transient

## Definition
```java
// Signature
public class DensityReturnType extends ReturnType {
```

## Architecture & Concepts
The DensityReturnType class is a fundamental component within the procedural world generation engine, acting as a conditional multiplexer for density functions. It is not a standalone service but rather a behavioral node within a larger density graph, typically configured declaratively in world generation assets.

Its primary architectural role is to enable **data-driven selection** of terrain or material generation logic. It evaluates a primary *choice* density function at a given sample point. Based on the resulting value, it selects a corresponding *sample* density function from a predefined map of value ranges.

This pattern is critical for creating complex, layered environmental features. For example:
*   **Geological Stratification:** The choice density could represent altitude. Different ranges of altitude values would then map to different sample densities representing layers of rock, dirt, or ore.
*   **Biome Blending:** The choice density could represent a temperature or humidity map, with different ranges selecting the appropriate biome-specific density functions.

The class also contains specialized logic for calculating `distanceFromWall`, a mechanism used to modify density calculations near the boundaries of procedural regions (e.g., Voronoi cells). This allows for sophisticated effects like smooth material transitions or the generation of unique features along biome borders.

## Lifecycle & Ownership
-   **Creation:** DensityReturnType instances are not intended for direct manual instantiation. They are constructed by the world generator's configuration loader when it parses the declarative generator graph from asset files (e.g., JSON).
-   **Scope:** An instance persists for the entire lifetime of its parent world generator. It is created once during the generator's initialization phase and is reused for all subsequent density calculations within that session.
-   **Destruction:** The object is eligible for garbage collection when the world generator itself is discarded, for instance, when a server shuts down or a single-player world is closed.

## Internal State & Concurrency
-   **State:** This class is **effectively immutable** post-construction. The constructor converts the input `Map<Range, Density>` into internal, final arrays for performance. These arrays are never modified during the object's lifetime. All other fields are final.

-   **Thread Safety:** The class is **conditionally thread-safe**. The `get` method does not mutate any internal state and allocates new objects (Context, Vector3d clones) for each call, preventing interference between threads. However, its overall thread safety is entirely dependent on the thread safety of the `Density` objects provided to its constructor.

    **WARNING:** If the `choiceDensity` or any of the `sampleDensities` are stateful and not designed for concurrent access, race conditions will occur when the world generator runs on multiple threads. All injected density functions must be stateless or properly synchronized.

## API Surface
The public contract is exclusively defined by the `get` method inherited from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(distance0, distance1, samplePoint, closestPoint0, closestPoint1, context) | double | O(N) | Calculates a final density value. N is the number of density ranges. This is the primary entry point for the class's logic. |

## Integration Patterns

### Standard Usage
This class is not invoked directly by gameplay code. It is a node within a density graph that is processed by a higher-level orchestrator, such as a `PositionNode`. The system retrieves a value from it as part of a larger world generation calculation.

```java
// Hypothetical orchestrator code
// A DensityReturnType instance is held by a parent node.
// The orchestrator calls 'get' with the current world coordinates and context.

// context is a pre-existing Density.Context for the current thread
// positionNode.returnType is an instance of DensityReturnType
double finalDensity = positionNode.returnType.get(
    dist0,
    dist1,
    samplePoint,
    cPoint0,
    cPoint1,
    context
);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new DensityReturnType()` in code. The world generator's behavior should be defined in configuration assets, not hard-coded. This ensures the system remains data-driven and flexible.
-   **Excessive Ranges:** Configuring this node with a very large number of `densityDelimiters` will create a performance bottleneck. The `get` method performs a linear scan over the ranges, leading to O(N) complexity. For high-performance generation, keep the number of ranges minimal.
-   **Overlapping Ranges:** The internal logic does not support overlapping ranges in the `densityDelimiters` map. The first range that matches the choice value will be selected. This can lead to non-deterministic or unexpected behavior if ranges are not defined discretely.

## Data Pipeline
The flow of data through a DensityReturnType instance is a multi-stage process that transforms a spatial query into a single density value.

> Flow:
> World Generator Request -> `get` method invoked with sample point & context
> 1.  The primary `choiceDensity` is processed to yield a `choiceValue`.
> 2.  The `choiceValue` is compared against the internal `delimiters` array in a linear scan.
> 3.  A matching range selects a corresponding `sampleDensity` function.
> 4.  A new, localized `Density.Context` is created, potentially enriched with `distanceFromCellWall` data.
> 5.  The selected `sampleDensity` is processed using this new context.
> 6.  The final density `double` is returned to the World Generator.

