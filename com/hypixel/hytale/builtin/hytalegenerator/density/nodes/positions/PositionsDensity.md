---
description: Architectural reference for PositionsDensity
---

# PositionsDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes.positions
**Type:** Transient

## Definition
```java
// Signature
public class PositionsDensity extends Density {
```

## Architecture & Concepts
The PositionsDensity class is a fundamental component within the procedural world generation engine. It operates as a specialized *Density Function*, a node in a larger graph that computes a scalar value (density) for any given 3D coordinate.

Its primary function is to calculate density based on proximity to a set of predefined points. It bridges a spatial query system, represented by the PositionProvider, with the density calculation pipeline. By querying for nearby points and processing their distances, it can generate complex, organic patterns like Voronoi diagrams, cellular noise, or fields of influence around specific locations.

This class is configured with four key strategies that define its behavior:
1.  **PositionProvider:** The source of truth for the point cloud. This provider is queried to find all relevant points within a search radius.
2.  **DistanceFunction:** A strategy that defines the metric for distance (e.g., Euclidean, Manhattan, Chebyshev). This allows for different visual textures in the generated patterns.
3.  **ReturnType:** A strategy that transforms the calculated distances (e.g., distance to the nearest point, difference between the two nearest points) into the final density value.
4.  **maxDistance:** A critical optimization parameter that defines the search radius, limiting the query space to prevent performance degradation.

## Lifecycle & Ownership
-   **Creation:** Instances of PositionsDensity are not created manually. They are instantiated by the world generation graph parser when loading a world generation configuration (e.g., from a JSON definition). It exists as a node within an immutable configuration graph.
-   **Scope:** The object's lifetime is tied to the loaded world generation configuration. It persists as long as that configuration is active and is reused for all density calculations within its scope.
-   **Destruction:** The object is eligible for garbage collection when the world generation configuration is unloaded or replaced. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The object is **effectively immutable**. All of its core dependencies (PositionProvider, ReturnType, DistanceFunction) and parameters are declared as final and are injected via the constructor. The object holds no mutable state between invocations of its process method.
-   **Thread Safety:** This class is **thread-safe**. The process method is fully re-entrant. All calculations and temporary state (e.g., closest points, calculated distances) are confined to local variables within the method's stack frame. A single PositionsDensity instance can be safely shared and invoked concurrently by multiple world generation worker threads.

## API Surface
The public contract is minimal, centered on the constructor for dependency injection and the core process method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PositionsDensity(provider, returnType, distanceFunc, maxDist) | constructor | O(1) | Constructs the density node. Throws IllegalArgumentException if maxDistance is negative. |
| process(Density.Context) | double | O(N) | Calculates the density at the context's position. Complexity is dependent on the number of points (N) returned by the PositionProvider within the search radius. |
| cellNoiseDistanceFunction(double) | static Double2DoubleFunction | O(1) | A static factory method providing a common distance-to-density mapping function used for cellular noise patterns. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, imperative use. It is designed to be a declarative node within a world generation graph. The following conceptual example illustrates how it would be defined in a configuration file.

```java
// A conceptual representation of how this class is used.
// The engine parses this configuration and constructs the object graph.

// 1. Define a source of points (e.g., random points for a forest)
PositionProvider treePositions = new PoissonDiskProvider(...);

// 2. Define how to calculate the final value
ReturnType nearestPointDistance = new ReturnNthDistance(0); // Return distance to nearest point

// 3. Define the distance metric
DistanceFunction euclidean = new EuclideanDistance();

// 4. Configure the PositionsDensity node
Density forestClearing = new PositionsDensity(
    treePositions,
    nearestPointDistance,
    euclidean,
    64.0 // Search radius
);

// 5. The engine then calls process() on the 'forestClearing' node
//    for millions of points during chunk generation.
double density = engine.calculateDensity(forestClearing, new Vector3d(x, y, z));
```

### Anti-Patterns (Do NOT do this)
-   **Unbounded Search:** Providing a very large maxDistance value can lead to catastrophic performance issues. The number of points returned by the PositionProvider can grow cubically with the search radius, making the process method a major bottleneck.
-   **Inefficient PositionProvider:** The performance of this node is entirely dependent on the efficiency of the injected PositionProvider. Using a provider that performs a linear scan instead of using a spatially-accelerated data structure (like an octree) will render this node unusable for large point sets.
-   **Stateful Dependencies:** Injecting stateful, non-thread-safe implementations for PositionProvider, ReturnType, or DistanceFunction will break the concurrency model and lead to race conditions and non-deterministic world generation.

## Data Pipeline
The flow of data through the process method is a multi-stage pipeline that transforms a 3D coordinate into a scalar density value.

> Flow:
> Density.Context (with query position) -> **PositionsDensity.process()**
> 1.  Calculate search bounding box using maxDistance.
> 2.  Invoke PositionProvider.positionsIn() with the bounding box.
> 3.  For each point returned:
>     -   Invoke DistanceFunction.getDistance().
>     -   Update internal list of two closest points.
> 4.  Invoke ReturnType.get() with final distances and closest points.
> 5.  Return final double (density value).

