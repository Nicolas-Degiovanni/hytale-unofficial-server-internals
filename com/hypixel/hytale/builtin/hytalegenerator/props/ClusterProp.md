---
description: Architectural reference for ClusterProp
---

# ClusterProp

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props
**Type:** Transient

## Definition
```java
// Signature
public class ClusterProp extends Prop {
```

## Architecture & Concepts

The ClusterProp is a composite propagator within the world generation system. It does not represent a single world feature, but rather orchestrates the placement of many other Prop instances in a spatially-defined group. Its primary function is to create natural-looking clusters of objects, such as forests, boulder fields, or patches of vegetation.

Architecturally, it acts as a meta-layer in the generation pipeline. It decouples the logic of *finding a suitable area for a group* from the logic of *what to place within that group and how*.

The core operational concept involves three phases:
1.  **Anchor Scanning:** A configured Scanner and Pattern are used to identify valid anchor points for the center of each cluster.
2.  **Density-based Distribution:** For each anchor point, the ClusterProp iterates over a square area defined by its range. At each coordinate within this area, it calculates the distance from the anchor. This distance is fed into a Double2DoubleFunction, the weightCurve, to determine a placement probability or density.
3.  **Child Propagation:** If a random check against the density succeeds, a child Prop is selected from a WeightedMap. The system then invokes the standard scan and place lifecycle on this child Prop, effectively delegating the final block placement.

This model allows designers to control the shape and density of clusters (via the weightCurve) and the composition of the cluster (via the WeightedMap) independently.

## Lifecycle & Ownership

-   **Creation:** A ClusterProp is a configuration object. It is instantiated directly via its constructor during the setup phase of the world generator, typically when defining the features for a specific biome. All dependencies, such as the child props, scanner, and density curve, are injected at this time.
-   **Scope:** The object's lifetime is tied to the world generation configuration. It is effectively a stateless blueprint that is used by generation worker threads. It does not persist beyond the execution of the world generation stage that utilizes it.
-   **Destruction:** The object is eligible for garbage collection as soon as the generator configuration is no longer referenced, typically after the world has been generated. It manages no native resources and has no explicit destruction method.

## Internal State & Concurrency

-   **State:** The ClusterProp is **immutable** after construction. All of its fields are final and are assigned exclusively in the constructor. It holds no mutable runtime state; its purpose is to encapsulate a generation algorithm and its configuration.

-   **Thread Safety:** This class is **thread-safe** and designed for concurrent execution by multiple world generation workers.
    -   The primary `place` method is re-entrant. It creates a new `FastRandom` instance for each cluster anchor point, seeded deterministically from the world coordinates. This critical pattern ensures that generation is both parallelizable and reproducible, preventing thread contention on a shared random number generator.
    -   All internal collections and configuration objects (propWeightedMap, pattern, scanner) are read-only after construction.
    -   The mutable world data, VoxelSpace and EntityContainer, are passed as arguments. The ClusterProp operates on these via a `WindowVoxelSpace`, a view that narrows its scope. This implies that the calling framework, the Staged Conveyor system, is responsible for partitioning the world space to prevent write conflicts between threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(position, materialSpace, id) | PositionListScanResult | O(S) | Uses the configured Scanner to find valid anchor points for clusters. Complexity S is determined by the injected Scanner implementation. |
| place(context) | void | O(N * R²) | Primary execution method. Iterates through anchor points from the scan result and orchestrates the placement of child props within the defined range R. |
| getContextDependency() | ContextDependency | O(1) | Returns the calculated read and write dependencies, crucial for the Staged Conveyor scheduler. |
| getWriteBounds() | Bounds3i | O(1) | Returns the maximum possible area this Prop can modify, relative to an anchor point. |

## Integration Patterns

### Standard Usage

A ClusterProp is not invoked manually. It is configured and registered with a higher-level generation system, such as a biome definition. The world generation engine is responsible for invoking the `scan` and `place` methods at the correct stage of the pipeline.

```java
// Example configuration during world generator setup
WeightedMap<Prop> forestProps = new WeightedMap<>();
forestProps.add(new OakTreeProp(), 10.0);
forestProps.add(new BirchTreeProp(), 3.0);
forestProps.add(new ForestFloorProp(), 25.0);

// A curve that makes clusters dense in the middle and sparse at the edges
Double2DoubleFunction falloffCurve = distance -> Math.max(0, 1.0 - (distance / 16.0));

Prop forestCluster = new ClusterProp(
    16,                      // range
    falloffCurve,            // weightCurve
    worldSeed + 1,           // seed
    forestProps,             // propWeightedMap
    new OnGroundPattern(),   // pattern
    new RandomGridScanner()  // scanner
);

// This 'forestCluster' prop would then be added to a biome's feature list.
// The engine handles the rest.
```

### Anti-Patterns (Do NOT do this)

-   **Using Child Props with Large Dependencies:** The constructor for ClusterProp intentionally filters out child props that have horizontal read or write dependencies. Attempting to bypass this can break the assumptions of the Staged Conveyor system, leading to generation artifacts or race conditions where a child prop tries to read data that has not been generated yet.
-   **Stateful Child Props:** The child props stored in the WeightedMap must also be thread-safe and preferably stateless. A stateful child Prop could introduce severe concurrency bugs when placed multiple times in parallel by different worker threads.
-   **Inefficient Curves or Ranges:** Using a very large range with a weightCurve that results in an extremely low placement probability is computationally wasteful. The system will iterate over a massive area only to place a handful of objects.

## Data Pipeline

The data flow for a ClusterProp is orchestrated by the world generation engine.

> Flow:
> World Generation Stage Start -> Engine calls **ClusterProp.scan()** with VoxelSpace -> Scanner produces List of anchor positions -> Engine calls **ClusterProp.place()** with anchors -> Inner loop iterates over area defined by *range* -> **weightCurve** and **FastRandom** determine placement -> **WeightedMap** selects a child Prop -> **childProp.place()** is called with a **WindowVoxelSpace** -> Child Prop writes Material data into the VoxelSpace.

