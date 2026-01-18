---
description: Architectural reference for Cylinder
---

# Cylinder

**Package:** com.hypixel.hytale.math.shape
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Cylinder implements Shape {
```

## Architecture & Concepts
The Cylinder class is a fundamental mathematical primitive representing an elliptical cylinder aligned with the Y-axis. It serves as a concrete implementation of the abstract **Shape** interface, placing it within the engine's polymorphic geometry system for spatial queries and volumetric calculations.

Architecturally, this class is designed as a lightweight, mutable data structure, not an encapsulated service. Its primary role is to define a volume in 3D space which can then be used by higher-level systems for tasks such as:
- Defining the area of effect for abilities or tools.
- Specifying volumes for world generation or modification routines.
- Performing hit detection or containment checks.

A key design decision is the delegation of complex voxel iteration logic. The Cylinder itself holds only its defining parameters (height, radii). The computationally intensive task of rasterizing this continuous mathematical shape into a discrete set of world blocks is handled by the **BlockCylinderUtil** utility class. This separates the data representation from the complex algorithmic implementation, promoting cleaner code and single responsibility.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically with a direct call to `new Cylinder(...)`. They are not managed by a dependency injection container or service registry. The expectation is that they are cheap to create and destroy.
- **Scope:** The lifecycle of a Cylinder object is almost always method-scoped. It is instantiated for a specific, immediate calculation and becomes a candidate for garbage collection once the operation completes. It is not designed to be a long-lived, persistent object.
- **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. There are no native resources or manual memory management concerns associated with this class.

## Internal State & Concurrency
- **State:** The Cylinder is a highly mutable object. Its core geometric properties—height, radiusX, and radiusZ—are public fields, allowing for direct, low-overhead modification. This design choice prioritizes performance and ease of use in tight game loops over strict encapsulation. Methods like `expand` further confirm its mutable nature.

- **Thread Safety:** **This class is not thread-safe.** Due to its public mutable fields and state-modifying methods, concurrent access from multiple threads without external synchronization will lead to race conditions and unpredictable behavior. A Cylinder instance should be confined to a single thread or protected by explicit locking if sharing is unavoidable.

## API Surface
The public API provides the core functionality for interacting with the cylinder's volume.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| containsPosition(x, y, z) | boolean | O(1) | Performs a fast mathematical check to determine if a continuous 3D point is inside the cylinder's volume. |
| forEachBlock(...) | boolean | O(r² * h) | Iterates over every discrete integer block coordinate that intersects the shape. Delegates to BlockCylinderUtil. |
| getBox(x, y, z) | Box | O(1) | Calculates the minimal Axis-Aligned Bounding Box (AABB) that fully encloses the cylinder at a given origin. |
| expand(radius) | void | O(1) | Modifies the internal state by increasing the X and Z radii. |

## Integration Patterns

### Standard Usage
The most common use case is to define a volume and then iterate over the blocks it contains to apply some logic. This is central to world editing tools, procedural generation, and gameplay mechanics.

```java
// Example: A spell that turns all blocks in a cylindrical area to stone.
Cylinder effectVolume = new Cylinder(10.0, 5.0, 5.0);
World world = Game.getWorld();
Vec3 targetPosition = player.getLookTarget();

// The forEachBlock method is the primary entry point for world interaction.
effectVolume.forEachBlock(targetPosition.x, targetPosition.y, targetPosition.z, 0.0, (x, y, z) -> {
    world.setBlock(x, y, z, Block.STONE);
    return true; // Return true to continue iterating.
});
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never share and modify a single Cylinder instance across multiple threads without implementing external locking. This will corrupt its state.
- **Stateful Reuse without Reset:** Reusing a Cylinder instance across different calculations without explicitly resetting its height and radii is dangerous. Because it is mutable, old values will persist and lead to incorrect results. Always create a new instance or re-assign all properties.

## Data Pipeline

The Cylinder class acts as an input for spatial query algorithms rather than a stage in a data processing pipeline. The flow of control is initiated by a game system, which uses the Cylinder to query the world.

> Flow:
> Game Logic (e.g., Spell System) -> Instantiates **Cylinder** with parameters -> Invokes `forEachBlock` -> Delegates to `BlockCylinderUtil` algorithm -> Executes provided callback for each intersecting block coordinate -> World State is mutated.

