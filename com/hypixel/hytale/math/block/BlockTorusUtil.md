---
description: Architectural reference for BlockTorusUtil
---

# BlockTorusUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockTorusUtil {
```

## Architecture & Concepts

BlockTorusUtil is a stateless, static utility class designed for high-performance volumetric iteration. Its sole responsibility is to calculate the block coordinates that form a torus (a donut-like shape) in 3D space and apply a user-defined operation to each coordinate.

This class acts as a foundational tool for procedural generation, large-scale world editing, and spell or ability effects. It abstracts the complex geometric calculations required to rasterize a continuous mathematical shape onto a discrete voxel grid. The core algorithm iterates a bounding box around the torus and performs a distance check for each point to determine if it lies within the shape.

A key implementation detail is the use of a floating-point adjustment constant (`0.41F`) added to the minor radius. This factor is a form of anti-aliasing for voxels; it ensures that blocks on the very edge of the mathematical torus are included, preventing gaps and creating a more solid, visually appealing shape.

The class uses a functional approach by accepting a `TriIntObjPredicate` as a consumer. This decouples the geometry calculation from the world-mutation logic, allowing callers to define arbitrary actions (e.g., placing blocks, checking for air, damaging entities) to be performed on the resulting coordinates.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, BlockTorusUtil is never instantiated. The Java Virtual Machine loads the class into memory at runtime, making its static methods available.
-   **Scope:** The class and its methods are available for the entire application lifetime.
-   **Destruction:** The class is unloaded by the JVM when the application terminates. No manual resource management is necessary.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** The class contains no member fields and its methods operate exclusively on the arguments provided. Each call is an independent, deterministic calculation.
-   **Thread Safety:** **Fully Thread-Safe.** The methods of BlockTorusUtil can be safely invoked from multiple threads simultaneously without external synchronization.
    **WARNING:** While the utility itself is thread-safe, the provided `consumer` predicate is not guaranteed to be. If the consumer modifies shared state (e.g., a world chunk), the caller is responsible for ensuring its thread safety.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(origin, outerR, minorR, context, consumer) | boolean | O((R+r)² * r) | Iterates every block coordinate within a solid torus. The loop is short-circuited if the consumer returns false. Throws IllegalArgumentException for non-positive radii. |
| forEachBlock(origin, outerR, minorR, thickness, capped, context, consumer) | boolean | O((R+r)² * r) | Iterates block coordinates within a hollow torus shell. **WARNING:** The `capped` parameter is currently unimplemented and has no effect on the calculation. |

## Integration Patterns

### Standard Usage

The primary pattern is to invoke a static method with a lambda or method reference that defines the operation per block. This is commonly used for world generation or modification tasks.

```java
// Example: Creating a ring of stone blocks
World world = ...;
BlockPos origin = new BlockPos(100, 64, 200);

// The consumer predicate places a block and always returns true to continue iteration.
BlockTorusUtil.forEachBlock(origin.x, origin.y, origin.z, 20, 5, world, (x, y, z, w) -> {
    w.setBlockState(new BlockPos(x, y, z), Block.STONE);
    return true; 
});
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The class has no public constructor and cannot be instantiated. Do not attempt `new BlockTorusUtil()`.
-   **Invalid Radii:** Providing a `minorRadius` or `outerRadius` of zero or less will result in an `IllegalArgumentException`. Always validate radius inputs.
-   **Ignoring Consumer Thread Safety:** Passing a stateful, non-synchronized consumer to this utility from multiple threads will introduce race conditions. The caller must manage the concurrency of the operation itself.

## Data Pipeline

BlockTorusUtil does not process a stream of data but rather generates it. It acts as a coordinate generator based on mathematical parameters, feeding these coordinates into a caller-defined sink (the consumer).

> Flow:
> Caller Input (Origin, Radii, Consumer) -> **BlockTorusUtil** (Geometry Calculation & Voxelization) -> Generated Block Coordinates -> Consumer Predicate Execution -> World State Mutation

