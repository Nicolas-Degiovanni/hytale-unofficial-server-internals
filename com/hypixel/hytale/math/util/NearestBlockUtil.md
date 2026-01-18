---
description: Architectural reference for NearestBlockUtil
---

# NearestBlockUtil

**Package:** com.hypixel.hytale.math.util
**Type:** Utility

## Definition
```java
// Signature
public final class NearestBlockUtil {
```

## Architecture & Concepts

NearestBlockUtil is a high-performance, stateless computational utility designed for a single purpose: to resolve a continuous, floating-point world coordinate to the nearest discrete, integer-based block coordinate that satisfies a specific condition. It forms a fundamental component of the bridge between continuous systems (like physics, player movement, and raycasting) and the discrete voxel grid of the game world.

The core algorithm is not a simple rounding operation. Instead, it calculates the squared Euclidean distance from the source point to the surface of the six adjacent block cells. This provides a geometrically accurate result for determining which block a point is "closest" to.

The utility's power lies in its decoupling of geometric calculation from game logic. This is achieved via a `BiPredicate` function passed into its main method. The caller provides the definition of a "valid" block, allowing this single utility to be used for a wide range of features, such as:
*   Finding the nearest solid ground for entity placement.
*   Determining the closest attachable surface for block placement previews.
*   Identifying the nearest interactive block for AI decision-making.

The `IterationElement` inner class defines the search candidates. The `DEFAULT_ELEMENTS` constant provides a standard search of the six face-adjacent blocks, which is sufficient for most use cases.

## Lifecycle & Ownership

As a static utility class, NearestBlockUtil does not have a lifecycle in the traditional sense of an object instance.

*   **Creation:** The class is never instantiated. Its private constructor explicitly throws an `UnsupportedOperationException` to enforce this pattern.
*   **Scope:** Static. The class and its methods are loaded by the JVM's ClassLoader and are available globally throughout the application's lifetime.
*   **Destruction:** The class is unloaded when its ClassLoader is garbage collected, which typically occurs only during application shutdown.

## Internal State & Concurrency

*   **State:** Stateless. The class contains no mutable static fields. The `DEFAULT_ELEMENTS` array is a compile-time constant and must be treated as immutable. All calculations are performed on local variables confined to the method's stack frame.
*   **Thread Safety:** This class is inherently and unconditionally thread-safe. Its methods are pure functions with respect to class state. They can be invoked from any number of concurrent threads without risk of race conditions or data corruption. No external locking or synchronization is required.

## API Surface

The public API consists of several static method overloads that all delegate to a single core implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findNearestBlock(elements, x, y, z, validBlock, t) | @Nullable Vector3i | O(N) | Core algorithm. Finds the nearest block to a coordinate, filtered by a predicate. N is the size of the elements array. Returns null if no valid block is found. |
| findNearestBlock(position, validBlock, t) | @Nullable Vector3i | O(1) | Convenience overload using `DEFAULT_ELEMENTS`. Complexity is constant time as it always iterates 6 elements. |

## Integration Patterns

### Standard Usage

This utility should be invoked directly via its static methods. The primary integration pattern involves providing a lambda or method reference for the `validBlock` predicate.

```java
// Example: Find the nearest surface a player can place a block on,
// using the World service to define validity.

Vector3d playerAimPoint = player.getAimTarget();
World world = context.getService(World.class);

// Define the condition for a valid placement target.
// The predicate receives a candidate block position and the world context.
BiPredicate<Vector3i, World> canPlaceOn = (pos, w) -> w.isBlockSolid(pos);

// Execute the search
Vector3i placementSurface = NearestBlockUtil.findNearestBlock(
    playerAimPoint,
    canPlaceOn,
    world
);

if (placementSurface != null) {
    // Render placement preview at 'placementSurface'
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Never attempt to create an instance with `new NearestBlockUtil()`. This will fail with an `UnsupportedOperationException` and violates the class design.
*   **Expensive Predicates:** The `validBlock` predicate is executed for every candidate block in the search set (typically 6 times). A predicate that performs slow operations, such as disk I/O, complex pathfinding, or heavy database queries, will create a significant performance bottleneck. Predicates should be fast, ideally performing simple lookups.
*   **Ignoring Null Return:** The `findNearestBlock` method can and will return null if no candidate block satisfies the predicate. Failure to perform a null check will result in `NullPointerException` errors in downstream code.

## Data Pipeline

NearestBlockUtil is a pure computational function, not a persistent stage in a data pipeline. It acts as a transformer within a larger logical flow.

> Flow:
> System (e.g., Physics, AI) -> `findNearestBlock` Call with Position & Predicate -> **NearestBlockUtil** -> `BiPredicate.test()` on World State -> System receives `Vector3i` or `null`

