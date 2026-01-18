---
description: Architectural reference for MortonCode
---

# MortonCode

**Package:** com.hypixel.hytale.component.spatial
**Type:** Utility

## Definition
```java
// Signature
public class MortonCode {
```

## Architecture & Concepts
The MortonCode class is a low-level, high-performance utility for spatial hashing. Its core function is to convert a three-dimensional coordinate (x, y, z) into a single 64-bit integer. This process, known as Z-order curve encoding, maps multi-dimensional data to one dimension while preserving spatial locality.

This is a foundational component for many of the engine's spatial partitioning and query systems, such as Octrees, k-d trees, or broad-phase physics culling. By representing object positions with a single, sortable value, the engine can perform highly efficient proximity queries, neighbor searches, and region-based lookups. The resulting Morton codes allow spatially adjacent objects to be located next to each other in memory, dramatically improving cache performance for world traversal and rendering algorithms.

The implementation uses a fixed precision of 21 bits per axis, mapping world coordinates within a specified bounding box to a discrete integer grid.

## Lifecycle & Ownership
As a pure utility class composed exclusively of static methods, MortonCode has no object lifecycle.

- **Creation:** The class is never instantiated. It cannot be created as an object.
- **Scope:** Its methods are globally accessible throughout the application's lifetime, loaded by the JVM's class loader.
- **Destruction:** The class is unloaded only when the JVM shuts down. No manual cleanup is ever required.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no member variables and all its methods are pure functions; their output depends solely on their input arguments.
- **Thread Safety:** MortonCode is inherently thread-safe. Since it holds no state and performs only mathematical computations on its inputs, it can be called concurrently from any number of threads without locks or synchronization primitives.

## API Surface
The public API consists of a single static method for encoding coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| encode(double x, double y, double z, double minX, double minY, double minZ, double maxX, double maxY, double maxZ) | long | O(1) | Normalizes and encodes a 3D coordinate into a 64-bit Morton code. Coordinates are clamped to the provided bounding box. |

## Integration Patterns

### Standard Usage
This class should be invoked directly whenever a 3D world position needs to be converted into a spatial hash for indexing or sorting purposes. It is a fundamental building block for higher-level spatial data structures.

```java
// Example: Calculating a Morton code for an entity's position
// within a specific world chunk/region.
BoundingBox regionBounds = world.getRegionBounds();
Vector3 position = entity.getPosition();

long spatialKey = MortonCode.encode(
    position.x, position.y, position.z,
    regionBounds.minX, regionBounds.minY, regionBounds.minZ,
    regionBounds.maxX, regionBounds.maxY, regionBounds.maxZ
);

// The 'spatialKey' can now be used to insert the entity
// into a spatially-aware data structure.
spatialMap.put(spatialKey, entity);
```

### Anti-Patterns (Do NOT do this)
- **Instantiation:** Attempting to create an instance via `new MortonCode()` is incorrect. The class is not designed to be instantiated and all methods are static.
- **Ignoring Bounding Box:** Passing arbitrary or unvalidated bounding box coordinates can lead to incorrect or poorly distributed keys. The quality of the spatial hashing is directly dependent on the accuracy of the world bounds provided.
- **Floating Point Precision:** Be aware that the input double coordinates are mapped to a 21-bit integer grid. Extreme precision beyond this resolution will be lost. Do not rely on Morton codes for exact geometric comparisons.

## Data Pipeline
The MortonCode class acts as a pure transformation stage in a larger data processing pipeline, typically for indexing world objects.

> Flow:
> 3D World Coordinate (Vector3) + BoundingBox -> **MortonCode.encode** -> 64-bit Spatial Key (long) -> Spatial Data Structure (e.g., Octree, Sorted List)

