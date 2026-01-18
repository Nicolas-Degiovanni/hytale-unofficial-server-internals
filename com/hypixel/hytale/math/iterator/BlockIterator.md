---
description: Architectural reference for BlockIterator
---

# BlockIterator

**Package:** com.hypixel.hytale.math.iterator
**Type:** Utility

## Definition
```java
// Signature
public final class BlockIterator {
```

## Architecture & Concepts

BlockIterator is a high-performance, stateless utility class designed for efficient voxel traversal. Its primary function is to execute a raycast through a 3D grid of unit-sized blocks and invoke a callback for each block intersected along the ray's path. This component is fundamental to systems requiring world intersection tests, such as player interaction, projectile physics, and AI line-of-sight calculations.

The core of BlockIterator is an implementation of a fast voxel traversal algorithm, conceptually similar to the Amanatides and Woo algorithm. It operates without allocating memory for a collection of results. Instead, it employs a callback-based design using the `BlockIteratorProcedure` functional interface. This pattern is critical for performance, as it avoids the overhead of creating and populating a list of intersected blocks, which is a significant cost in performance-sensitive code that runs every frame.

The system is designed with floating-point precision in mind, utilizing a nested `FastMath` class with custom equality checks to handle the inherent inaccuracies of floating-point arithmetic in a deterministic way.

## Lifecycle & Ownership
- **Creation:** Not applicable. BlockIterator is a final utility class with a private constructor that throws `UnsupportedOperationException`. It is never instantiated.
- **Scope:** Stateless. The scope of any operation is confined to the duration of a single static method call. The class itself holds no state and therefore has no lifecycle.
- **Destruction:** Not applicable. As no instances are created, no garbage collection or manual cleanup is required.

## Internal State & Concurrency
- **State:** Stateless. All methods are pure functions whose output depends solely on their input arguments. The class holds no mutable or immutable fields, and each invocation is entirely independent.
- **Thread Safety:** Inherently thread-safe. Due to its stateless nature, BlockIterator can be safely called from any number of concurrent threads without risk of race conditions or data corruption. No locks or synchronization primitives are required or used.

**WARNING:** While the BlockIterator itself is thread-safe, the user-provided `BlockIteratorProcedure` callback is not. The caller is responsible for ensuring that the logic within the callback is thread-safe if the iteration is initiated from a multi-threaded context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| iterateFromTo(origin, target, procedure) | boolean | O(N) | Traverses all blocks on a line between two points. N is the number of blocks along the path. Returns false if the procedure aborted the iteration. |
| iterate(origin, direction, maxDistance, procedure) | boolean | O(N) | Traverses all blocks along a ray from an origin. N is the number of blocks traversed up to maxDistance. Returns false if the procedure aborted the iteration. |
| BlockIteratorProcedure.accept(...) | boolean | O(1) | The user-defined callback. Receives block coordinates and intersection points. Return **false** to abort the iteration; **true** to continue. |
| BlockIteratorProcedurePlus1.accept(...) | boolean | O(1) | A generic variant of the callback that accepts a user-provided object, avoiding lambda captures in performance-critical code. |

## Integration Patterns

### Standard Usage
The primary pattern is to provide a lambda or method reference as the `BlockIteratorProcedure`. The procedure should contain the logic to be executed for each block and return `false` to terminate the traversal early, which is a common optimization.

```java
// Example: Find the first solid block within 10 meters of the camera's view.
Vector3d cameraPos = getCameraPosition();
Vector3d lookVector = getCameraLookVector();
final AtomicReference<Vector3i> firstSolidBlock = new AtomicReference<>();

// The iterate method returns true if it completed, false if aborted.
boolean wasAborted = !BlockIterator.iterate(cameraPos, lookVector, 10.0,
    (blockX, blockY, blockZ, entryX, entryY, entryZ, exitX, exitY, exitZ) -> {
        if (world.isBlockSolid(blockX, blockY, blockZ)) {
            firstSolidBlock.set(new Vector3i(blockX, blockY, blockZ));
            return false; // Block found, stop iterating.
        }
        return true; // Not solid, continue to the next block.
    }
);

if (wasAborted) {
    System.out.println("Found solid block at: " + firstSolidBlock.get());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance of BlockIterator, even via reflection. The class is not designed to be instantiated.
- **Long-Running Callbacks:** The callback is executed synchronously within a performance-critical loop. Do not perform heavy computations, file/network I/O, or acquire contended locks inside the `accept` method. Doing so will block the calling thread and can cause severe performance degradation.
- **Ignoring Return Value:** The boolean return value of `iterate` and `iterateFromTo` indicates whether the traversal was aborted by the callback. Ignoring this can lead to incorrect assumptions about whether the full distance was checked.

## Data Pipeline
BlockIterator is not a data pipeline in the traditional sense; it is a procedural control-flow mechanism. The flow represents the transfer of control and immediate data between the caller and the utility.

> Flow:
> Caller (e.g., Physics Engine, Player Controller) -> **BlockIterator.iterate()** -> [Internal Voxel Traversal Loop] -> User-defined `BlockIteratorProcedure` -> Boolean (continue/stop) -> **BlockIterator.iterate()** -> Caller receives final boolean status.

