---
description: Architectural reference for BlockCylinderUtil
---

# BlockCylinderUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockCylinderUtil {
```

## Architecture & Concepts

BlockCylinderUtil is a stateless, procedural geometry utility designed for rasterizing volumetric shapes onto the game's discrete block grid. It serves as a foundational tool for world generation, building tools, and spell effects that require creating or modifying cylindrical or elliptical-cylinder regions of blocks.

The core architectural concept is the decoupling of shape calculation from the action performed. The utility calculates the integer coordinates of every block that falls within the specified cylinder volume. For each coordinate, it invokes a caller-provided callback, a functional interface of type TriIntObjPredicate. This implementation of the **Visitor Pattern** allows a single, highly optimized geometry algorithm to be used for a wide variety of tasks, such as placing blocks, deleting blocks, or querying block data, without modification to the utility itself.

A notable implementation detail is the use of a `0.41F` adjustment to the radii. This constant is used to improve the voxelization aesthetic, ensuring the resulting shape more closely matches the intended continuous volume by correctly handling blocks at the boundary of the ellipse.

## Lifecycle & Ownership

-   **Creation:** This class is never instantiated. As a utility class containing only static methods, its functionality is accessed directly via the class name.
-   **Scope:** The methods are globally accessible and their lifetime is tied to the JVM's ClassLoader. They are available for the entire duration of the application session.
-   **Destruction:** Not applicable. There are no instances to manage or destroy.

## Internal State & Concurrency

-   **State:** This utility is completely **stateless**. All computations are performed exclusively on the arguments provided to its static methods. It does not maintain any internal state, caching, or session data between calls.

-   **Thread Safety:** The class is inherently **thread-safe**. Its stateless nature ensures that concurrent calls from multiple threads will not interfere with one another.

    **WARNING:** While the utility itself is thread-safe, the provided TriIntObjPredicate consumer is not guaranteed to be. If the consumer modifies a shared resource (e.g., adds block coordinates to a global list, modifies world state directly), the caller is responsible for ensuring the consumer's thread safety through external locking or by using thread-safe data structures.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(origin..., radiusX, height, radiusZ, t, consumer) | boolean | O(radiusX * radiusZ * height) | Iterates over every block position within a **solid** elliptical cylinder. Returns false if the consumer aborts the loop. |
| forEachBlock(origin..., thickness, t, consumer) | boolean | O(radiusX * radiusZ * height) | Iterates over every block position within a **hollow** elliptical cylinder of a given thickness. |
| forEachBlock(origin..., thickness, capped, t, consumer) | boolean | O(radiusX * radiusZ * height) | Iterates over a hollow cylinder, with an option to include solid top and bottom caps. |

## Integration Patterns

### Standard Usage

The primary pattern is to invoke a static method with a lambda expression or method reference acting as the consumer. The generic object `t` is used to pass context, such as a world handle or a block palette, into the consumer without resorting to capturing variables in the lambda.

```java
// How a developer should normally use this
WorldEditor editor = context.getWorldEditor();
BlockPos origin = new BlockPos(100, 64, 200);
int radius = 8;
int height = 16;

// Create a solid stone cylinder
BlockCylinderUtil.forEachBlock(
    origin.getX(), origin.getY(), origin.getZ(),
    radius, height, radius,
    editor,
    (x, y, z, worldEditor) -> {
        worldEditor.setBlock(x, y, z, Block.STONE);
        return true; // Continue iteration
    }
);
```

### Anti-Patterns (Do NOT do this)

-   **Attempted Instantiation:** Do not attempt to create an instance of BlockCylinderUtil. It is a static utility and cannot be instantiated.
-   **Stateful, Non-Concurrent Consumers:** Passing a stateful consumer that is not thread-safe (e.g., `(x, y, z, list) -> list.add(new BlockPos(x,y,z))`) into a parallelized world generation system can cause race conditions and data corruption. All shared state modified by the consumer must be managed with concurrency in mind.
-   **Ignoring Return Value:** The boolean return value indicates whether the entire volume was processed. If the consumer can abort early (e.g., upon finding a specific block), ignoring the return value can lead to incorrect assumptions about the operation's completeness.

## Data Pipeline

BlockCylinderUtil acts as a generator in a data or control-flow pipeline. It does not transform data itself but rather generates a stream of coordinates that are fed to a subsequent processing stage defined by the consumer.

> Flow:
> World Generator / Game Logic -> **BlockCylinderUtil.forEachBlock**(params) -> Generates (x, y, z) Coordinates -> Consumer Lambda -> World State Mutation / Data Collection

