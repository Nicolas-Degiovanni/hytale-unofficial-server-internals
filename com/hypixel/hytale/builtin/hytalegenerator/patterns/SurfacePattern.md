---
description: Architectural reference for SurfacePattern
---

# SurfacePattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class SurfacePattern extends Pattern {
```

## Architecture & Concepts
The SurfacePattern is a composite structural component within the world generation framework. It does not represent a single block or entity, but rather acts as a high-level layout strategy for arranging other, simpler patterns. Its primary function is to define a geometric formation consisting of two parallel, circular disks of points in 3D space and to test if child patterns can be placed at those points.

This class embodies the **Composite** design pattern. It combines two delegate Pattern objects, a *wallPattern* and an *originPattern*, into a more complex structure.

Operationally, SurfacePattern heavily front-loads its computational work into the constructor. Upon instantiation, it calculates two distinct sets of relative 3D coordinates:
1.  **surfacePositions:** A disk of points representing a floor, ceiling, or wall.
2.  **originPositions:** A smaller, concentric disk of points, typically for a central feature.

These point clouds are then rotated and translated according to the specified Facing and gap parameters. The final bounding box for the entire composite shape is also pre-calculated and cached. This pre-computation is a critical optimization, ensuring that subsequent calls to the matches method are highly performant, involving only iteration and delegation rather than complex trigonometric or distance calculations.

## Lifecycle & Ownership
-   **Creation:** A SurfacePattern is instantiated directly via its public constructor, typically during the initialization phase of a higher-level world generation feature like a dungeon or biome-specific structure. It is not managed by a dependency injection framework or service locator.
-   **Scope:** The object's lifetime is ephemeral and tied to the specific generation task that created it. It is designed to be configured once, used repeatedly for matching against multiple world locations, and then discarded.
-   **Destruction:** The object is eligible for garbage collection as soon as the parent generator that holds a reference to it completes its work and is itself collected. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state of a SurfacePattern is **effectively immutable** after the constructor completes. It maintains final references to the delegate patterns and holds two pre-computed lists of Vector3i coordinates. While the Vector3i objects themselves are mutated *during* construction to apply facing transformations, they are not modified by any other public method.
-   **Thread Safety:** This class is **thread-safe** for all read operations, primarily matches and readSpace. As its state is fixed post-construction, a single SurfacePattern instance can be safely shared and used by multiple parallel world generation worker threads without requiring external locks or synchronization. This assumes that the composed *wallPattern* and *originPattern* objects are also thread-safe.

## API Surface
The public contract is minimal, adhering to the base Pattern interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Context context) | boolean | O(N+M) | Evaluates if the composed patterns match the world at all pre-calculated relative coordinates. N and M are the number of points in the surface and origin disks, respectively. Returns false on the first mismatch. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-computed bounding box that fully encloses the entire pattern. This is a fast, constant-time cache lookup. |

## Integration Patterns

### Standard Usage
A SurfacePattern is used as a configuration object to define a complex structural feature. It is created once and then used by a generator to test for valid placement locations.

```java
// Define the component patterns for the floor and a central feature
Pattern floorPattern = new BlockPattern(Blocks.STONE_BRICKS);
Pattern featurePattern = new BlockPattern(Blocks.GLOWSTONE);

// Configure the SurfacePattern to create a circular stone brick floor
// with a glowstone feature at its center, oriented as a floor (Facing.U).
SurfacePattern roomFloor = new SurfacePattern(
    floorPattern,       // The pattern for the disk surface
    featurePattern,     // The pattern for the center
    15.0,               // Radius of the stone brick floor
    1.0,                // Radius of the glowstone feature
    SurfacePattern.Facing.U, // "Up" - creates a horizontal floor below the origin
    1,                  // Gap: places the floor 1 block below the context position
    0                   // Gap: places the feature at the context position
);

// A world generator would then use this pattern to check for placement
if (roomFloor.matches(currentGenerationContext)) {
    // The area is clear; proceed to place the floor and feature
    // by applying the patterns.
}
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in Loops:** Do not create a new SurfacePattern inside a tight loop, such as for every block in a chunk. The constructor is computationally expensive and is designed to be called once during setup.
-   **Large Radii:** Using excessively large radii can lead to significant memory consumption and performance degradation. The number of positions stored grows with the square of the radius, which can cause the constructor to become very slow and the matches method to perform thousands of checks.

## Data Pipeline
The SurfacePattern acts as a complex predicate or filter within the world generation data flow. It does not transform data itself but rather determines whether a subsequent generation step should proceed.

> Flow:
> World Generation Task -> Feature Generator -> **SurfacePattern.matches(context)** -> [true/false] -> Voxel Placement Stage

