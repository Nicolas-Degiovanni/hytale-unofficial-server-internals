---
description: Architectural reference for Bounds3i
---

# Bounds3i

**Package:** com.hypixel.hytale.builtin.hytalegenerator.bounds
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class Bounds3i implements MemInstrument {
```

## Architecture & Concepts

The Bounds3i class is a fundamental geometric primitive representing an Axis-Aligned Bounding Box (AABB) in a discrete, three-dimensional integer space. It is a cornerstone for spatial reasoning within the engine, particularly for systems operating on a grid, such as world generation, physics, and spatial partitioning.

Its core design is defined by two Vector3i points: *min* and *max*. A critical and intentional design choice is that the volume defined is inclusive of the *min* coordinate and **exclusive** of the *max* coordinate. This convention simplifies iteration and prevents common off-by-one errors when processing voxel grids. For a volume of size (1,1,1), *min* would be (x,y,z) and *max* would be (x+1, y+1, z+1).

The class is architected to be **highly mutable**, with most manipulation methods like `offset` and `encompass` modifying the object's state in-place and returning a reference to `this` to enable method chaining. To ensure the integrity of the bounding box, an internal `correct` method is invoked after construction and any state-altering assignment. This function guarantees that the *min* vector's components are always less than or equal to the corresponding *max* vector's components, making the object's state robust and predictable.

## Lifecycle & Ownership

-   **Creation:** Bounds3i instances are transient and created on-demand. They are typically instantiated at the beginning of a calculation, such as defining a query area for a spatial database or calculating the bounding volume of a game asset. There is no central factory or manager.

-   **Scope:** The lifetime of a Bounds3i object is typically short and confined to the scope of the method in which it was created. They are frequently used as temporary variables for intermediate calculations.

-   **Destruction:** As a standard Java object, instances are automatically marked for garbage collection once they are no longer referenced. No manual memory management is required.

## Internal State & Concurrency

-   **State:** The class is **mutable**. Although the `min` and `max` fields are declared `final`, they are references to mutable Vector3i objects. The internal state of these vectors is modified directly by most of the class's public methods. Defensive copies are made in the constructor to prevent external objects from affecting the internal state after creation.

-   **Thread Safety:** **This class is not thread-safe.** Its mutable design makes it inherently unsafe for concurrent access without external synchronization. If multiple threads modify the same Bounds3i instance, the internal state will become corrupted, leading to race conditions and unpredictable behavior. Any system sharing Bounds3i instances across threads must implement its own locking mechanism or, preferably, pass copies created via the `clone` method.

## API Surface

The public API is designed for efficient, in-place geometric manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| contains(Vector3i position) | boolean | O(1) | Checks if a point is within the volume. |
| intersects(Bounds3i other) | boolean | O(1) | Performs a fast AABB intersection test. |
| encompass(Bounds3i other) | Bounds3i | O(1) | Mutates this bound to expand and include the other. |
| assign(Bounds3i other) | Bounds3i | O(1) | Mutates this bound to become a copy of the other. |
| clone() | Bounds3i | O(1) | Creates a new, deep-copy instance of the bound. |
| correct() | void | O(1) | Enforces the internal invariant that min <= max. |

## Integration Patterns

### Standard Usage

The intended use is for short-lived, single-threaded geometric calculations. When passing a bound to another system that may modify it, always provide a clone to prevent side effects.

```java
// Create a primary region for a structure
Bounds3i structureBounds = new Bounds3i(new Vector3i(10, 20, 10), new Vector3i(20, 30, 20));

// Define a query area
Bounds3i queryArea = new Bounds3i(new Vector3i(15, 0, 15), new Vector3i(30, 40, 30));

// Check for intersection before performing expensive work
if (structureBounds.intersects(queryArea)) {
    // To avoid side-effects, work with a clone of the original bounds
    Bounds3i workingCopy = structureBounds.clone();
    workingCopy.offset(new Vector3i(0, 5, 0)); // Modify the copy
    // ... continue processing
}
```

### Anti-Patterns (Do NOT do this)

-   **Shared Mutable State:** Never pass a single Bounds3i instance to multiple subsystems or threads that might modify it. This is the most common source of errors. Use the `clone()` method to provide each system with its own copy.

-   **Direct Field Manipulation:** While the `min` and `max` fields are public, directly manipulating their components (e.g., `myBounds.min.setX(50)`) is dangerous. This bypasses the `correct()` logic that is automatically called by methods like `assign`, potentially leaving the bounds in an invalid state where a `min` component is greater than a `max` component.

-   **Ignoring Max-Exclusivity:** Forgetting that the `max` coordinate is an exclusive boundary will lead to off-by-one errors, especially during iteration. A loop should run from `min.x` to `max.x - 1`.

## Data Pipeline

Bounds3i is not a data processing component itself, but rather a data structure that defines scope and context within larger pipelines. It is used to specify regions of interest for processing.

**Example: World Generation Pipeline**

> Flow:
> Biome Placement Logic -> **Bounds3i (Defines Biome Area)** -> Structure Generation System -> Voxel Data Output

**Example: Rendering Culling Pipeline**

> Flow:
> Entity Transform -> **Bounds3i (Entity Bounding Box)** -> Frustum Intersection Test -> Render Command Emitter

