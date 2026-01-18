---
description: Architectural reference for BlockSphereUtil
---

# BlockSphereUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockSphereUtil {
```

## Architecture & Concepts

BlockSphereUtil is a high-performance, stateless computational utility designed for volumetric block iteration. It serves as a foundational mathematics library for any system that needs to identify or operate on blocks within a spherical or ellipsoidal region. Its primary consumers are world generation, large-scale block modification systems (e.g., explosions, terraforming spells), and certain AI or physics queries.

The core design is centered on decoupling the iteration logic from the operation performed on each block. This is achieved by accepting a `TriIntObjPredicate` functional interface as a callback. The utility calculates the integer coordinates of each block within the specified volume and invokes the predicate for each one. This allows the caller to define a custom action—such as changing a block type, adding an entity, or performing a collision check—without modifying the complex iteration algorithm.

The algorithms are heavily optimized:
1.  **Ellipsoid Equation:** The iteration is based on solving the standard ellipsoid equation for integer coordinates, ensuring mathematically correct boundaries.
2.  **Symmetry Exploitation:** The `forEachBlock` methods that do not involve thickness calculate results for a single octant and then mirror those results across the other seven octants. This reduces the number of expensive square root and multiplication operations by a factor of eight. This optimization is handled internally by the private `test` method.
3.  **Approximation for Quality:** The integer-based `forEachBlock` methods add a small constant (0.41F) to the radius. This is a deliberate approximation to counteract the aliasing effects of discretizing a smooth sphere onto a block grid. It ensures that blocks lying on the very edge of the sphere are included, resulting in a visually fuller and more natural shape.

## Lifecycle & Ownership

BlockSphereUtil is a static utility class and as such, it is never instantiated.

-   **Creation:** The class is loaded into the JVM by the ClassLoader when first referenced by another component. No instance is ever created.
-   **Scope:** The class and its static methods are available globally for the entire application lifetime once loaded.
-   **Destruction:** The class is unloaded only when its defining ClassLoader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency

-   **State:** BlockSphereUtil is **completely stateless**. It contains no member fields and all computations are based exclusively on the arguments passed into its static methods. All variables are method-scoped.

-   **Thread Safety:** The class is **inherently thread-safe**. As it holds no state, multiple threads can call any of its methods concurrently without risk of race conditions or data corruption.

    **WARNING:** While the utility itself is thread-safe, the provided `TriIntObjPredicate` consumer may not be. Callers are responsible for ensuring that the logic within their consumer is thread-safe if the iteration is invoked from a multi-threaded context. For example, modifying a shared `List` within the consumer requires external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlockExact(origin, radius, t, consumer) | void | O(r³) | Iterates all blocks within a sphere defined by a high-precision double radius. Throws `IllegalArgumentException` if radius is not positive. |
| forEachBlock(origin, radius, t, consumer) | void | O(r³) | Overload for a simple sphere using an integer radius. Internally calls the ellipsoid variant. |
| forEachBlock(origin, radii, t, consumer) | boolean | O(rx\*ry\*rz) | Iterates all blocks within an ellipsoid. Returns false if the consumer predicate returns false, short-circuiting the loop. |
| forEachBlock(origin, radii, thickness, t, consumer) | boolean | O(r³ - (r-t)³) | Iterates blocks within a hollow ellipsoid (a shell). Returns false if the consumer predicate returns false, short-circuiting the loop. |

## Integration Patterns

### Standard Usage

The primary pattern is to invoke a static method with a lambda expression or method reference that implements the desired block operation. This is common in systems that need to modify a region of the world.

```java
// Find all stone blocks within a 10-block radius of the player
// and store their positions for later processing.

List<BlockPos> stoneBlocks = new ArrayList<>();
BlockPos playerPos = ...;
World world = ...;

// The consumer predicate checks the block type at the given coordinates.
// It returns true to continue iteration, or false to stop.
TriIntObjPredicate<World> stoneFinder = (x, y, z, w) -> {
    if (w.getBlockAt(x, y, z).isType("stone")) {
        stoneBlocks.add(new BlockPos(x, y, z));
    }
    return true; // Continue searching
};

BlockSphereUtil.forEachBlock(
    playerPos.getX(),
    playerPos.getY(),
    playerPos.getZ(),
    10,
    world,
    stoneFinder
);
```

### Anti-Patterns (Do NOT do this)

-   **Expensive Callbacks:** Do not perform blocking or computationally expensive operations (e.g., file I/O, network requests, complex pathfinding) inside the consumer predicate. The predicate is called hundreds or thousands of times for even a small radius, and any delay will severely impact the performance of the calling thread.

-   **Ignoring Short-Circuiting:** The `boolean` return type on some `forEachBlock` methods is a critical feature for optimization. If you are searching for a single block or condition, return `false` from the consumer as soon as it is found. Failing to do so forces the utility to iterate over the entire volume unnecessarily.

-   **Unsafe Concurrent Modification:** Do not modify a non-thread-safe collection or object from within the consumer if `BlockSphereUtil` is being called from multiple threads. The utility guarantees its own safety, not the safety of the code it is executing.

## Data Pipeline

BlockSphereUtil acts as a data **generator**, not a pipeline stage. It transforms input parameters into a stream of 3D integer coordinates, which are then fed directly into a user-provided processing function.

> Flow:
> Input Parameters (Origin, Radius, Thickness) -> **BlockSphereUtil Algorithm** -> Stream of (x, y, z) Coordinates -> `TriIntObjPredicate` Consumer -> Caller-Defined Side Effects (e.g., World State Change, Data Collection)

