---
description: Architectural reference for IChunkBounds
---

# IChunkBounds

**Package:** com.hypixel.hytale.server.worldgen.util.bounds
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IChunkBounds {
```

## Architecture & Concepts
The IChunkBounds interface is a foundational contract within the server's world generation system. It defines the abstract concept of a two-dimensional, axis-aligned bounding box (AABB) in world block coordinates. Its primary purpose is to provide a standardized, immutable representation of a spatial area, decoupling world generation algorithms from the concrete implementation of their boundaries.

This interface is not a component with behavior, but rather a data-oriented contract. Systems like prefab placement, biome distribution, and feature generation rely on IChunkBounds to perform efficient spatial queries. By operating against this interface, different generation stages can reason about areas of influence without needing to know the origin or specific shape of those areas. For example, a structure placer can use an IChunkBounds to quickly determine which chunks it overlaps, enabling targeted modifications rather than broad, inefficient scans.

The interface is heavily utilized for:
-   Defining the footprint of prefabs and structures.
-   Calculating the area of effect for a biome or terrain feature.
-   Constraining random point selection to a specific region.
-   Performing broad-phase intersection tests between different world generation elements.

## Lifecycle & Ownership
As an interface, IChunkBounds itself has no lifecycle. The following describes the lifecycle of objects that *implement* this interface.

-   **Creation:** Instances are typically created on-demand by higher-level world generation coordinators or when a specific asset, like a prefab, is loaded and its spatial footprint needs to be calculated. They are value objects, created to represent a specific area for a specific task.
-   **Scope:** The scope is almost always transient and task-specific. An IChunkBounds object usually exists only for the duration of a single, focused generation operation (e.g., placing one structure, calculating one biome region). It is not a long-lived, session-scoped object.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection as soon as the generation task that created them is complete and all references are dropped.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. However, any class implementing IChunkBounds is expected to hold the four integer boundaries that define the AABB: low X, low Z, high X, and high Z. Best practice, and the assumption made by the engine's multithreaded world generator, is that implementing objects are **immutable**. Their boundary values should be finalized upon construction.
-   **Thread Safety:** The interface is inherently thread-safe as all provided default methods are pure functions that do not modify internal state. They compute new values based on the object's boundaries and the supplied arguments.

    **WARNING:** If a custom implementation of IChunkBounds is created with mutable state, it is the responsibility of the implementer to ensure thread safety. Mutable bounds are a significant anti-pattern and will lead to severe and difficult-to-debug race conditions in the parallelized world generation pipeline.

## API Surface
The primary contract consists of four abstract methods to define the box. All other methods are default utilities built upon this core contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLowBoundX() | int | O(1) | Returns the minimum X coordinate (inclusive). |
| getLowBoundZ() | int | O(1) | Returns the minimum Z coordinate (inclusive). |
| getHighBoundX() | int | O(1) | Returns the maximum X coordinate (inclusive). |
| getHighBoundZ() | int | O(1) | Returns the maximum Z coordinate (inclusive). |
| intersectsChunk(long) | boolean | O(1) | Performs an AABB intersection test against the bounds of a single chunk. Critical for performance. |
| getLowBoundX(PrefabRotation) | int | O(1) | Calculates the new minimum X coordinate after applying a rotation to the bounds. |
| randomX(Random) | int | O(1) | Returns a pseudo-random X coordinate within the bounds. |
| getLowChunkX() | int | O(1) | Converts the minimum block X coordinate to its corresponding chunk X coordinate. |

## Integration Patterns

### Standard Usage
IChunkBounds is used as a parameter or a local variable within a generation algorithm to define its area of operation. The most common pattern is to check for intersection with the current chunk being processed before performing expensive work.

```java
// A hypothetical feature placer operating on a specific chunk
void placeFeatureInChunk(int chunkX, int chunkZ, IChunkBounds featureBounds) {
    // First, perform a cheap check to see if the feature can even exist in this chunk.
    if (!featureBounds.intersectsChunk(chunkX, chunkZ)) {
        return; // Do nothing, the feature is entirely outside this chunk.
    }

    // If it intersects, proceed with more complex logic, like finding a random position.
    Random random = new Random();
    int worldX = featureBounds.randomX(random);
    int worldZ = featureBounds.randomZ(random);

    // ... logic to place the feature at (worldX, worldZ)
}
```

### Anti-Patterns (Do NOT do this)
-   **Mutable Implementations:** Never create an implementation of IChunkBounds where the boundary values can change after construction. This violates the contract's implicit assumption of immutability and will break concurrent world generation.
-   **Block-by-Block Iteration:** Do not use the bounds to iterate over every single block coordinate within the area. The interface is designed for broad-phase spatial queries, not fine-grained iteration.
    ```java
    // BAD: Incredibly inefficient for large bounds
    for (int x = bounds.getLowBoundX(); x <= bounds.getHighBoundX(); x++) {
        // ...
    }
    ```
-   **Ignoring Rotations:** When dealing with placeable assets like prefabs, failing to use the rotation-aware `get...Bound...(rotation)` methods will result in incorrect boundary calculations for any non-default orientation.

## Role in World Generation Pipeline
IChunkBounds is not a processing stage in a pipeline but rather a fundamental data structure used *by* the pipeline stages for spatial reasoning. It carries essential metadata that informs the decision-making process of various generators.

> Flow:
> Prefab Definition -> **IChunkBounds Creation** -> Structure Placement Algorithm -> **IChunkBounds::intersectsChunk** check -> Block Writing to Chunk Buffer

