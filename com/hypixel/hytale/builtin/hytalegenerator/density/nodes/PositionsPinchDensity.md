---
description: Architectural reference for PositionsPinchDensity
---

# PositionsPinchDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class PositionsPinchDensity extends Density {
```

## Architecture & Concepts
The PositionsPinchDensity is a spatial deformation node within the procedural world generation framework. It does not generate density values itself; instead, it manipulates the coordinate space for a downstream *input* Density function. Its primary function is to create localized warping effects, such as pinching, pulling, or pushing terrain around specific points of interest.

Conceptually, this node operates as a local warp field generator. When the generator queries for the density at a given sample point, this class first intercepts the request. It queries a PositionProvider to find all relevant "attractor" or "repulsor" points within a defined radius (maxDistance). For each nearby point, it calculates a displacement vector. The magnitude and direction of this displacement are governed by a user-provided Double2DoubleFunction, the pinchCurve.

The final warped sample coordinate is a weighted average of the displacements from all nearby points. This new, displaced coordinate is then used to sample the input Density. This mechanism is fundamental for creating organic features where one element influences the shape of another, for example, a crystal formation pushing surrounding rock outwards or a cave entrance pulling the terrain inwards.

## Lifecycle & Ownership
-   **Creation:** An instance of PositionsPinchDensity is created and configured during the initialization of a world generator. It is a node within a larger, static graph of Density objects that collectively define the generation algorithm. It is not managed by a dependency injection container but rather composed directly by the generator's configuration.
-   **Scope:** The object's lifetime is tied to the world generator instance it belongs to. It persists as long as the generator is in memory and is used across multiple generation tasks for a given world.
-   **Destruction:** The object is eligible for garbage collection when the parent world generator is destroyed. It holds no unmanaged resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is stateful. Its behavior is defined by its configured fields: input, positions, pinchCurve, maxDistance, and distanceNormalized. However, this state is established at creation and is treated as immutable during the density processing phase. The process method does not modify the internal state of the PositionsPinchDensity instance itself.
-   **Thread Safety:** This class is conditionally thread-safe. The instance itself can be safely shared and read by multiple worker threads. The core process method is re-entrant as it does not modify its own fields.

    **WARNING:** The process method **mutates the passed-in Density.Context object**. It is imperative that each worker thread operates on its own unique instance of Density.Context to prevent race conditions and ensure deterministic output. The world generation engine is expected to enforce this pattern.

## API Surface
The public contract is focused on configuration and the primary processing function inherited from the Density superclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(N * M) | Calculates a warped sample position and returns the result of its input Density at that new position. N is the number of points returned by the PositionProvider; M is the complexity of the input Density. |
| setInputs(Density[] inputs) | void | O(1) | Configures the downstream input Density node. This is part of the graph-building contract. |

## Integration Patterns

### Standard Usage
PositionsPinchDensity is used as a wrapper around another Density node to apply a spatial transformation. It is never a terminal node in the graph.

```java
// Assume 'baseTerrain' is a Density function defining mountains and valleys.
// Assume 'caveEntrances' is a PositionProvider locating cave mouths.

// A curve that pulls space inwards, strongest at the center (distance=0).
Double2DoubleFunction pullCurve = distance -> (1.0 - distance) * -10.0;

// Create the pinch node to deform the base terrain around caves.
Density terrainNearCaves = new PositionsPinchDensity(
    baseTerrain,
    caveEntrances,
    pullCurve,
    20.0, // maxDistance
    true  // distanceNormalized
);

// When terrainNearCaves.process() is called, it will sample 'baseTerrain'
// at a location pulled towards the nearest cave entrance.
```

### Anti-Patterns (Do NOT do this)
-   **Missing Inputs:** Calling process without a configured input Density or PositionProvider will cause the node to act as a pass-through or return zero, nullifying its purpose.
-   **Unbounded Pinch Curve:** Providing a pinchCurve that returns excessively large values can cause the sampling coordinate to be displaced by extreme amounts. This can lead to sampling artifacts, aliasing, or pulling data from completely unrelated regions of the world.
-   **Excessive Max Distance:** Using a very large maxDistance can severely impact performance, as the PositionProvider will be queried over a large volume and the number of influencing points to process will increase dramatically.

## Data Pipeline
The flow of data is a transformation of spatial coordinates, not a direct manipulation of density values.

> Flow:
> Initial Sample Coordinate -> **PositionsPinchDensity** queries PositionProvider -> List of Nearby Points -> Warp Vectors Calculated & Averaged -> Final Warped Coordinate -> Input Density.process(Warped Coordinate) -> Final Density Value

