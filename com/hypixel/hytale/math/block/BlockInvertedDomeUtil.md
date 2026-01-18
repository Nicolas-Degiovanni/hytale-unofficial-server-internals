---
description: Architectural reference for BlockInvertedDomeUtil
---

# BlockInvertedDomeUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockInvertedDomeUtil {
```

## Architecture & Concepts
BlockInvertedDomeUtil is a stateless, high-performance procedural shape generator. Its sole responsibility is to calculate the integer coordinates of all blocks that fall within an inverted three-dimensional ellipsoid (a dome shape) and invoke a callback for each one. This component is a fundamental building block for world generation, structure placement, and large-scale block modification tools.

The core algorithm is an optimized implementation of the standard ellipsoid equation: `(x/rX)^2 + (y/rY)^2 + (z/rZ)^2 <= 1`. To maximize performance, the utility pre-calculates the inverse of the squared radii. It then iterates through a single octant of the ellipsoid and uses symmetry to derive the coordinates for the other three XZ-plane quadrants, significantly reducing the number of required calculations.

A constant floating-point value, `0.41F`, is added to each radius. This is a common voxel-fitting heuristic designed to ensure that the generated shape more accurately covers the centers of the blocks at its edge, producing a fuller, more visually appealing curve and preventing single-block gaps.

The "inverted" nature of this utility means that all `y` offsets from the origin are negative or zero, making it ideal for generating concave shapes like craters, bowls, or the underside of domes.

### Lifecycle & Ownership
- **Creation:** Not applicable. BlockInvertedDomeUtil is a static utility class and is never instantiated. Its methods are accessed directly via the class name.
- **Scope:** Application-level. The class is loaded by the JVM's class loader and its static methods are available for the entire duration of the application's runtime.
- **Destruction:** Not applicable. The class is unloaded only when its class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** **Stateless**. This class contains no member fields and stores no data between invocations. All computations are derived exclusively from the arguments provided to its static methods.

- **Thread Safety:** **Inherently Thread-Safe**. As a stateless utility, this class can be safely called by multiple threads concurrently without any risk of race conditions or data corruption *within the utility itself*.

    **WARNING:** The overall operation's thread safety is entirely dependent on the implementation of the provided `TriIntObjPredicate` consumer and the generic context object `T`. If the consumer modifies a shared resource (e.g., the game world) without proper synchronization, race conditions will occur. The caller is responsible for ensuring the safety of the callback logic.

## API Surface
The public contract consists of two overloaded methods for generating solid or hollow inverted domes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(originX, originY, originZ, radiusX, radiusY, radiusZ, t, consumer) | boolean | O(Rx\*Ry\*Rz) | Iterates over every block in a solid inverted ellipsoid. Returns false if the consumer predicate returns false, halting iteration. |
| forEachBlock(originX, originY, originZ, radiusX, radiusY, radiusZ, thickness, capped, t, consumer) | boolean | O(Rx\*Ry\*Rz) | Iterates over blocks in a hollow inverted ellipsoid shell of a given thickness. If capped is true, the flat top at y=originY is filled. |

## Integration Patterns

### Standard Usage
This utility is designed to be called by higher-level systems like world generators or gameplay logic that need to place blocks in a specific shape. The standard pattern involves passing a lambda expression or method reference as the consumer.

```java
// Standard usage for creating a 10x5x10 stone crater
// The consumer directly modifies the world state.
World world = context.getWorld();
BlockPos origin = new BlockPos(100, 64, 200);

BlockInvertedDomeUtil.forEachBlock(
    origin.getX(),
    origin.getY(),
    origin.getZ(),
    10, 5, 10, // Radii
    world, // Context object passed to the consumer
    (x, y, z, w) -> {
        // The consumer predicate sets a block and returns true to continue.
        w.setBlock(x, y, z, BlockType.STONE);
        return true;
    }
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new BlockInvertedDomeUtil()`. This is a static-only utility class and cannot be instantiated.
- **Main Thread Blocking:** Avoid calling this utility with very large radii on the main game thread. The iteration is synchronous and can block the thread, causing significant frame rate drops or server tick lag. For large operations, delegate the call to a separate worker thread or asynchronous task.
- **Unsafe Consumers:** Do not pass a consumer that modifies shared state from multiple threads without explicit synchronization. While the utility is thread-safe, your callback logic may not be.

## Data Pipeline
BlockInvertedDomeUtil does not process a stream of data. Instead, it acts as a generator in a control flow, translating a set of parameters into a series of function calls.

> Flow:
> World Generation System -> **BlockInvertedDomeUtil.forEachBlock()** -> [Internal Ellipsoid Calculation] -> Consumer Callback -> World State Modification API

