---
description: Architectural reference for ContextDependency
---

# ContextDependency

**Package:** com.hypixel.hytale.builtin.hytalegenerator.conveyor.stagedconveyor
**Type:** Value Object

## Definition
```java
// Signature
public class ContextDependency {
```

## Architecture & Concepts

The ContextDependency class is a fundamental data structure within the Staged Conveyor world generation system. It serves as a spatial contract, defining the precise data footprint required by a single world generation pass or stage. Its primary purpose is to enable the generator to reason about and aggregate the data dependencies of a complex, multi-stage pipeline *before* execution.

This class mathematically represents the relationship between the data a generator stage reads and the data it writes. By encapsulating these **read** and **write** ranges, it allows the Conveyor system to calculate the total bounding box of world data that must be loaded, allocated, and kept in memory for a sequence of operations to complete successfully.

The core concept is **stacking**. The `stackOver` method combines the dependency of a subsequent stage with a preceding one, correctly propagating the spatial requirements through the pipeline. This prevents out-of-bounds data access and allows for significant optimization by fetching all required world data in a single operation.

All dependency calculations are performed on the XZ plane; the `update` method, called during construction, systematically zeroes out the Y-component of all ranges. This reflects a world generation strategy that operates on vertical columns of data, where horizontal dependencies are the primary concern for planning.

### Lifecycle & Ownership
-   **Creation:** Instantiated by individual generator stages to declare their spatial needs. A `CaveGenerator`, for example, would create a ContextDependency instance defining how far it needs to read existing terrain data to carve out its tunnels. The static `EMPTY` instance serves as a null object or an initial value for aggregations.
-   **Scope:** Transient. These objects typically have a short lifecycle, existing only during the planning phase of the world generation conveyor. They are passed to a planner or manager which aggregates them into a final, total dependency.
-   **Destruction:** Managed by the Java garbage collector. Once the final generation bounds are calculated, the individual ContextDependency objects are no longer referenced and are reclaimed.

## Internal State & Concurrency
-   **State:** This class is designed to be **effectively immutable**. The constructor defensively clones the incoming `Vector3i` objects, and all public getters also return clones. This prevents both external and internal state mutation after the object has been created. The derived ranges (trashRange, externalDependencyRange) are calculated once at construction time.
-   **Thread Safety:** Inherently **thread-safe**. Due to its immutable nature, a single ContextDependency instance can be safely read and used by multiple threads simultaneously without locks or synchronization. All static methods are also pure functions that produce new instances without modifying their inputs, making them safe for concurrent use.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ContextDependency(read, write) | constructor | O(1) | Creates a dependency contract. Throws if inputs are null. |
| getTotalPropBounds_voxelGrid() | Bounds3i | O(1) | Calculates the total bounding box required for an operation, encompassing both read and write areas. |
| stackOver(other) | ContextDependency | O(1) | **CRITICAL:** Combines this dependency (the "over" stage) with another (the "under" stage) to produce a new, aggregate dependency. Order is significant. |
| getReadRange() | Vector3i | O(1) | Returns a clone of the vector defining the required input data radius. |
| getWriteRange() | Vector3i | O(1) | Returns a clone of the vector defining the modified output data radius. |
| getRequiredPadOf(dependencies) | static Vector3i | O(N) | Aggregates a list of dependencies to find the total external padding required. |
| stackMaps(under, over) | static Map | O(N+M) | Merges two maps of dependencies, stacking values where keys overlap. |

## Integration Patterns

### Standard Usage
A generator stage defines its dependency, which is then combined by a higher-level system to determine the total work area.

```java
// Stage 1: A "BaseTerrain" generator writes a 1-block border.
ContextDependency baseTerrainDep = new ContextDependency(
    new Vector3i(0, 0, 0), // Reads nothing
    new Vector3i(1, 0, 1)  // Writes a 1-block border
);

// Stage 2: A "Decorator" reads the terrain and its border to place trees.
ContextDependency decoratorDep = new ContextDependency(
    new Vector3i(1, 0, 1), // Reads the 1-block border
    new Vector3i(0, 0, 0)  // Writes nothing (for this example)
);

// The Conveyor system stacks the dependencies in execution order.
// The decorator runs *over* the base terrain.
ContextDependency totalDependency = decoratorDep.stackOver(baseTerrainDep);

// The result is a single dependency that can be used to allocate a voxel grid
// large enough for both operations to run safely.
```

### Anti-Patterns (Do NOT do this)
-   **Mutating Returned Vectors:** The getter methods return *clones* of the internal vectors. Modifying a returned vector will have no effect on the ContextDependency object itself. This is by design to ensure immutability.

    ```java
    // BAD - This does nothing
    ContextDependency dep = new ContextDependency(v1, v2);
    dep.getReadRange().set(10, 0, 10); // Modifies a temporary clone, not the object
    ```
-   **Incorrect Stacking Order:** The `stackOver` operation is not commutative. The order represents the execution order of the generation pipeline. `A.stackOver(B)` means stage A runs *after* stage B. Reversing the order will produce an incorrect total dependency.
-   **Assuming Y-Axis Relevance:** Do not rely on the Y-component of the input vectors for dependency calculation. The system is designed for 2D (XZ plane) spatial planning and will zero out the Y-axis internally.

## Data Pipeline
ContextDependency is a metadata component that *defines* the data requirements for a pipeline, rather than being a processing step within it.

> Flow:
> Generator Stage Definition → **ContextDependency Instance** → Conveyor Planner → `stackOver()` Aggregation → Final `Bounds3i` → Voxel Grid Allocation → Generator Execution on Grid

