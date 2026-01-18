---
description: Architectural reference for CellWallDistanceDensity
---

# CellWallDistanceDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Utility

## Definition
```java
// Signature
public class CellWallDistanceDensity extends Density {
```

## Architecture & Concepts
The CellWallDistanceDensity class is a fundamental leaf node within the procedural world generation's density function graph. It does not perform any calculations itself. Instead, its sole responsibility is to act as an accessor, retrieving a pre-computed value from the active generation context.

This component serves as a standardized input provider for more complex density functions. By encapsulating the access to the `distanceFromCellWall` property, it allows other nodes in the graph to declaratively depend on this value without needing knowledge of the underlying `Density.Context` structure. This is critical for creating modular and reusable generator definitions, where composite density functions can be built by wiring together primitive inputs like this one.

In the context of cellular noise (e.g., Voronoi or Worley noise), this value is essential for creating biome boundaries, procedural patterns, and feature placement logic that respects cell borders.

## Lifecycle & Ownership
- **Creation:** Instances of CellWallDistanceDensity are typically instantiated by a deserializer when loading world generator presets from asset files. They may also be created programmatically by higher-level generator logic that dynamically constructs density graphs.
- **Scope:** The object is transient and stateless. Its lifetime is tied to the lifetime of the parent density function graph that contains it. It holds no resources and can be garbage collected as soon as the graph is no longer referenced.
- **Destruction:** Managed entirely by the Java Garbage Collector. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is entirely determined by the `Density.Context` object passed into its `process` method.
- **Thread Safety:** This class is **inherently thread-safe**. The `process` method is a pure function relative to the object's own state. Multiple threads can safely call `process` on a shared instance of CellWallDistanceDensity, provided that each thread supplies its own thread-local `Density.Context`. The world generation engine is responsible for ensuring the thread safety of the context object itself.

## API Surface
The public API consists of a single method inherited and implemented from the `Density` base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the pre-calculated distance to the nearest cell wall for the given context. Throws NullPointerException if the context is null. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is designed to be a component within a larger, composite `Density` function, typically defined in a JSON or other asset format. The world generator's evaluation engine will invoke its `process` method during the density calculation for a specific block or region.

A conceptual example of its usage within a parent function:

```java
// PSEUDOCODE: How another Density function would use this
public class TerrainHeightDensity extends Density {
    private final Density baseHeight;
    private final Density cellWallDistance; // An instance of CellWallDistanceDensity

    // ... constructor ...

    @Override
    public double process(Density.Context context) {
        // Get the distance from the cell wall
        double wallDist = this.cellWallDistance.process(context);

        // Modify terrain height based on proximity to the wall
        if (wallDist < 0.1) {
            return 0; // Create a trench at the boundary
        }
        return this.baseHeight.process(context);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Misuse Outside Generation:** Do not attempt to instantiate and use this class outside of the world generator's density evaluation loop. The `process` method is meaningless without a valid and fully populated `Density.Context` provided by the generator.
- **Assumption of Scale:** The raw value returned by this node is not guaranteed to be normalized. Do not assume it is in a 0-1 range unless the specific generator implementation guarantees it. Higher-level density functions are responsible for remapping or clamping this value as needed.

## Data Pipeline
This class is a simple pass-through component in the data pipeline. It extracts a single field from a larger data object.

> Flow:
> World Generator -> Creates `Density.Context` -> **CellWallDistanceDensity.process(context)** -> Returns `context.distanceFromCellWall` -> Consumed by Parent Density Function

