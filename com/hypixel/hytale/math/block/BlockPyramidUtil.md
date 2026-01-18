---
description: Architectural reference for BlockPyramidUtil
---

# BlockPyramidUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockPyramidUtil {
```

## Architecture & Concepts

BlockPyramidUtil is a stateless, procedural geometry utility that provides highly optimized algorithms for iterating over block coordinates within a pyramidal volume. It is a foundational component for any system that needs to programmatically generate or query pyramid-like structures in the game world, such as world generation, structure placement, or spell effects.

The core design philosophy is the decoupling of iteration logic from the action performed. Instead of returning a large collection of block positions, which would be memory-intensive, the class employs a functional approach using a `TriIntObjPredicate` consumer. This consumer is invoked for each coordinate within the calculated volume. This pattern allows the caller to define the work to be done per-block, while the utility handles the complex boundary calculations.

A key feature is the consumer's boolean return value, which enables efficient short-circuiting. If the consumer returns false at any point, the entire iteration process is immediately terminated, preventing unnecessary computation.

## Lifecycle & Ownership

-   **Creation:** This class is never instantiated. As a utility class composed exclusively of static methods, it is loaded into the JVM by the class loader upon its first reference.
-   **Scope:** Application-level. Its methods are globally accessible throughout the application's lifetime.
-   **Destruction:** The class is unloaded by the JVM when the application terminates. There is no manual cleanup or destruction required.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** BlockPyramidUtil holds no internal state. All necessary data is provided as arguments to its static methods. Each method call is an independent, deterministic calculation.
-   **Thread Safety:** **Fully Thread-Safe.** Due to its stateless nature, all methods in BlockPyramidUtil can be safely called from multiple threads concurrently without any risk of race conditions or data corruption within the utility itself.

    **WARNING:** The overall thread safety of an operation depends entirely on the implementation of the provided consumer and the context object it captures. If the consumer modifies shared state without proper synchronization, the caller is responsible for ensuring thread safety.

## API Surface

The API provides methods for iterating over both solid and hollow pyramids, in both upright and inverted orientations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(origin..., t, consumer) | void | O(radiusX * radiusZ * height) | Iterates all block coordinates within a solid, upright pyramid. Throws IllegalArgumentException for non-positive dimensions. |
| forEachBlock(origin..., thickness, capped, t, consumer) | void | O(radiusX * radiusZ * height) | Iterates coordinates for a hollow pyramid shell. The capped parameter fills the top-most layers defined by the thickness. |
| forEachBlockInverted(origin..., t, consumer) | void | O(radiusX * radiusZ * height) | Iterates all block coordinates within a solid, inverted (point-down) pyramid. |
| forEachBlockInverted(origin..., thickness, capped, t, consumer) | void | O(radiusX * radiusZ * height) | Iterates coordinates for a hollow, inverted pyramid shell. |

## Integration Patterns

### Standard Usage

The primary use case is to apply an operation, such as placing blocks, to a pyramidal volume within the world. This is achieved by passing a lambda or method reference as the consumer.

```java
// Example: Build a solid stone pyramid in the world
World world = context.getWorld();
int originX = 100, originY = 64, originZ = 200;
int height = 30;
int radius = 15;

// The consumer lambda places a stone block at each coordinate.
// It always returns true to ensure the entire pyramid is processed.
BlockPyramidUtil.forEachBlock(originX, originY, originZ, radius, height, radius, world, (x, y, z, w) -> {
    w.setBlock(x, y, z, Block.STONE);
    return true; 
});
```

### Anti-Patterns (Do NOT do this)

-   **Expensive Consumers:** The consumer function is executed for every single block within the volume, potentially thousands or millions of times. Avoid performing heavyweight operations like file I/O, network requests, or complex pathfinding inside the consumer, as this will block the calling thread for an extended period.
-   **Ignoring Consumer Return:** Failing to use the `false` return value for early exit in search operations is inefficient. If you are looking for a specific condition, return `false` as soon as it is met to stop the iteration.
-   **Unsafe Concurrent Modification:** Do not use a consumer that modifies a shared, non-thread-safe collection or object from multiple threads without external locking. While BlockPyramidUtil is thread-safe, the state you modify within the consumer may not be.

## Data Pipeline

BlockPyramidUtil acts as a coordinate generator within a larger data flow. It does not transform data itself but rather drives a subsequent process by providing a stream of spatial coordinates.

> Flow:
> World Generator / Structure Placer -> **BlockPyramidUtil.forEachBlock** -> (x, y, z) Coordinates -> Consumer Lambda -> World State Mutation (e.g., World.setBlock)

