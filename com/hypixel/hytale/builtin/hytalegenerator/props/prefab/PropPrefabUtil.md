---
description: Architectural reference for PropPrefabUtil
---

# PropPrefabUtil

**Package:** com.hypixel.hytale.builtin.hytalegenerator.props.prefab
**Type:** Utility

## Definition
```java
// Signature
public class PropPrefabUtil {
```

## Architecture & Concepts
PropPrefabUtil is a stateless, static utility class designed to perform geometric calculations on prefab data structures. It acts as a high-level mathematical abstraction over the raw, memory-efficient `PrefabBufferAccessor`, providing a clean and centralized API for querying a prefab's spatial properties.

Its primary role within the engine is to support the world generation and prop placement systems. Before a prefab is instantiated and its blocks are written into the world, systems need to understand its physical footprint—its bounding box, size, and anchor point. This class provides that critical information, enabling collision detection, placement validation, and alignment calculations.

A key architectural function is its handling of rotations. The bounding box of a non-cubical prefab changes depending on its orientation. PropPrefabUtil correctly computes the axis-aligned bounding box (AABB) for any given `PrefabRotation`, abstracting away the complex rotational transformations from the calling systems.

## Lifecycle & Ownership
- **Creation:** As a static utility class, PropPrefabUtil is never instantiated. Its methods are invoked directly on the class definition. The class is loaded into the JVM by the class loader when first referenced.
- **Scope:** Application-wide. The class and its static methods are available for the entire duration of the server or client session.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no manual cleanup or destruction process.

## Internal State & Concurrency
- **State:** PropPrefabUtil is completely **stateless**. It contains no member variables and all its methods are pure functions; their output depends exclusively on their input arguments.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, multiple threads can invoke its methods concurrently without any risk of data corruption or race conditions. No synchronization mechanisms such as locks are necessary.

## API Surface
The public API consists of static methods for querying prefab dimensions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMin(prefab) | Vector3i | O(1) | Calculates the minimum corner of the prefab's AABB with zero rotation. |
| getMax(prefab) | Vector3i | O(1) | Calculates the maximum corner of the prefab's AABB with zero rotation. |
| getMin(prefab, rotation) | Vector3i | O(1) | Calculates the minimum corner of the prefab's AABB for a given rotation. |
| getMax(prefab, rotation) | Vector3i | O(1) | Calculates the maximum corner of the prefab's AABB for a given rotation. |
| getSize(prefab) | Vector3i | O(1) | Returns the dimensions (width, height, depth) of the prefab's AABB. |
| getAnchor(prefab) | Vector3i | O(1) | Retrieves the prefab's defined anchor point, used as its origin for placement. |

## Integration Patterns

### Standard Usage
This class should be used whenever a system needs to reason about the spatial characteristics of a prefab before it is placed in the world. The typical use case involves retrieving a prefab's data accessor and then using this utility to calculate its bounds for placement checks.

```java
// Assume 'prefabAccessor' is a valid PrefabBuffer.PrefabBufferAccessor instance
// obtained from a prefab management system.

// Calculate the dimensions of the prefab
Vector3i prefabSize = PropPrefabUtil.getSize(prefabAccessor);

// Determine the bounding box if the prefab were rotated 90 degrees
PrefabRotation rotation = PrefabRotation.ROTATION_90;
Vector3i minBounds = PropPrefabUtil.getMin(prefabAccessor, rotation);
Vector3i maxBounds = PropPrefabUtil.getMax(prefabAccessor, rotation);

// Use these values to check if the prefab fits in a designated area
boolean canPlace = world.checkCollision(minBounds, maxBounds);
```

### Anti-Patterns (Do NOT do this)
- **Object Instantiation:** Never attempt to create an instance of this class with `new PropPrefabUtil()`. It provides no value, as all methods are static.
- **Redundant Calculation:** Avoid calling these methods repeatedly for the same prefab within a performance-sensitive loop. The calculations are fast, but the results are deterministic. If the bounds of a specific prefab are needed multiple times, cache the resulting Vector3i object instead of re-calculating it.

## Data Pipeline
PropPrefabUtil is not a data transformer but rather a query service within a larger data flow, typically world generation. It provides metadata that informs subsequent stages of the pipeline.

> Flow:
> World Generation Algorithm -> Prefab Selection Logic -> Retrieves `PrefabBufferAccessor` -> **PropPrefabUtil** (calculates bounds) -> Placement Validation Logic -> World Block Buffer Update

