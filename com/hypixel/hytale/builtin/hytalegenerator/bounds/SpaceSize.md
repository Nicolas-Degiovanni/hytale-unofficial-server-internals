---
description: Architectural reference for SpaceSize
---

# SpaceSize

**Package:** com.hypixel.hytale.builtin.hytalegenerator.bounds
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class SpaceSize {
```

## Architecture & Concepts
The SpaceSize class is a foundational data structure representing a 3D, axis-aligned bounding box (AABB) using integer coordinates. It is primarily used within the world generation system to define regions, calculate spatial relationships between generated features, and manage the bounds of procedural structures.

Its core design principle is the use of a minimum *inclusive* coordinate and a maximum *exclusive* coordinate. This is a common and efficient pattern for spatial calculations, as it simplifies iteration and range checks. For example, a loop iterating over the x-axis would run from `min.x` up to, but not including, `max.x`.

**WARNING:** This class is annotated as **Deprecated**. It should not be used for new development and is likely superseded by a more robust, and potentially immutable, bounding box implementation elsewhere in the engine. Its continued presence is for legacy compatibility with older world generation modules. The mutable nature of this class is a significant design flaw that can lead to subtle and difficult-to-diagnose bugs.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the `new` keyword or through static factory methods like `merge` or `empty`. It is a plain Java object with no external dependencies or registry.
- **Scope:** SpaceSize objects are intended to be short-lived. They are typically created as temporary values within the scope of a single world generation algorithm or geometric calculation and discarded afterward.
- **Destruction:** Instances are managed by the Java Garbage Collector and are eligible for cleanup as soon as they are no longer referenced. There are no native resources or manual cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The `moveBy` method directly modifies the internal Vector3i fields. This is highly problematic, as an instance shared between multiple systems can be unexpectedly altered, leading to state corruption. While getter methods defensively return clones of the internal vectors, the SpaceSize object itself is not protected from modification.
- **Thread Safety:** This class is **Not Thread-Safe**. The mutable state is not protected by any synchronization mechanisms. Sharing a SpaceSize instance across multiple threads without external locking will result in race conditions and unpredictable behavior. It is designed for single-threaded use within a generation task.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SpaceSize(min, max) | Constructor | O(1) | Creates a bounding box from a minimum inclusive and maximum exclusive vector. |
| moveBy(delta) | SpaceSize | O(1) | **Mutates the object.** Translates the box by the given delta. Returns itself for chaining. |
| getMinInclusive() | Vector3i | O(1) | Returns a clone of the minimum inclusive corner of the box. |
| getMaxExclusive() | Vector3i | O(1) | Returns a clone of the maximum exclusive corner of the box. |
| clone() | SpaceSize | O(1) | Creates a new SpaceSize instance with the same bounds. |
| merge(a, b) | static SpaceSize | O(1) | Creates a new SpaceSize that fully encloses both input boxes. |
| stack(a, b) | static SpaceSize | O(1) | Creates a new SpaceSize representing the bounds of stacking one box on another. |
| empty() | static SpaceSize | O(1) | Creates a new, zero-sized SpaceSize at the origin. |

## Integration Patterns

### Standard Usage
The class is used to define and combine spatial regions during procedural generation.

```java
// How a developer should normally use this
// Define the bounds for two separate features
SpaceSize featureA = new SpaceSize(new Vector3i(0, 0, 0), new Vector3i(10, 5, 10));
SpaceSize featureB = new SpaceSize(new Vector3i(8, 0, 8), new Vector3i(18, 5, 18));

// Calculate the total bounding box required for both features
SpaceSize combined = SpaceSize.merge(featureA, featureB);

// The 'combined' object can now be used to allocate a region for generation.
```

### Anti-Patterns (Do NOT do this)
- **Shared Mutable State:** The most severe anti-pattern is sharing a single SpaceSize instance and modifying it, which creates dangerous side effects.

```java
// DANGEROUS: Do not do this
SpaceSize bounds = new SpaceSize(new Vector3i(0,0,0), new Vector3i(10,10,10));
RegionGenerator gen1 = new RegionGenerator(bounds);
RegionPainter painter1 = new RegionPainter(bounds);

// This call will unexpectedly change the bounds used by painter1
gen1.translateRegion(new Vector3i(5,0,0)); // Internally calls bounds.moveBy()
```

- **Continued Use:** Do not use this class in any new code. Find its modern, non-deprecated replacement. Relying on deprecated APIs creates technical debt and risks incompatibility with future engine updates.

## Data Pipeline
SpaceSize does not process data itself; it is a data structure that defines the boundaries for other processes. It typically appears at the beginning of a generation pipeline to scope the work of subsequent systems.

> Flow:
> Procedural Algorithm -> **Creates SpaceSize** -> Voxel Generation Task -> Voxel Buffer -> World Storage

In this flow, the SpaceSize object informs the Voxel Generation Task what region of the world it is responsible for filling.

