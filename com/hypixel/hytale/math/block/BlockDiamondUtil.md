---
description: Architectural reference for BlockDiamondUtil
---

# BlockDiamondUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockDiamondUtil {
```

## Architecture & Concepts

BlockDiamondUtil is a stateless, static utility class designed for high-performance, volumetric iteration. Its primary function is to provide algorithms for visiting every block within a three-dimensional diamond (or more accurately, an octahedron) shape centered at a given origin.

This class is a fundamental building block for systems that perform spatial operations, such as procedural generation, world editing tools, or area-of-effect spell systems. It decouples the complex logic of iterating a specific geometric shape from the action performed on each block within that shape. This is achieved by employing a functional pattern, accepting a **TriIntObjPredicate** as a callback. The caller provides the "what" (the operation), and BlockDiamondUtil handles the "where" (the diamond-shaped volume).

The internal algorithm is heavily optimized for performance. Instead of iterating through a bounding box and checking if each point is inside the diamond, it mathematically derives the bounds for each horizontal slice of the shape. Furthermore, it calculates positions for only one octant of the volume and uses symmetry to derive the corresponding positions in the other seven octants, significantly reducing computational overhead.

## Lifecycle & Ownership

-   **Creation:** This class is never instantiated. As a static utility, its methods are accessed directly via the class name.
-   **Scope:** The class and its methods are available for the entire application lifetime after being loaded by the Java Virtual Machine. It has no instance-specific scope or state.
-   **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable**. BlockDiamondUtil contains no member fields and holds no state between method calls. All required data is provided as method arguments.
-   **Thread Safety:** **Fully Thread-Safe**. Because the class is stateless, its methods can be safely called by multiple threads concurrently without any risk of race conditions or data corruption within the utility itself.

    **WARNING:** While the utility is thread-safe, the overall safety of an operation depends entirely on the provided **TriIntObjPredicate** consumer. If the consumer modifies shared state (e.g., world data), it must be responsible for its own synchronization.

## API Surface

The public API consists of two overloaded static methods for iterating over solid or hollow diamond volumes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(originX, originY, originZ, radiusX, radiusY, radiusZ, t, consumer) | static boolean | O(N) | Iterates every block in a solid diamond volume. N is proportional to radiusX * radiusY * radiusZ. Returns false if the consumer short-circuits the operation. |
| forEachBlock(originX, ..., thickness, capped, t, consumer) | static boolean | O(N) | Iterates every block in a hollow diamond shell of a given thickness. Returns false if the consumer short-circuits the operation. |

## Integration Patterns

### Standard Usage

The intended use is to call the static methods directly, providing a lambda or method reference for the consumer predicate. This is ideal for systems that need to modify or query blocks in a diamond shape.

```java
// Example: Fill a diamond shape with stone blocks in the world.
World world = ...;
BlockPos origin = new BlockPos(100, 64, 200);

// The predicate returns true to continue iteration, false to stop.
boolean success = BlockDiamondUtil.forEachBlock(
    origin.getX(), origin.getY(), origin.getZ(),
    10, 15, 10, // Radii for X, Y, Z
    world,
    (x, y, z, ctx) -> {
        ctx.setBlock(x, y, z, Block.STONE);
        return true; // Continue to next block
    }
);

if (!success) {
    // The operation was halted prematurely by the predicate.
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not attempt to create an instance with `new BlockDiamondUtil()`. This is a static utility class and cannot be instantiated.
-   **Expensive Predicates:** The consumer predicate is invoked for every single block within the volume, potentially thousands or millions of times. Avoid performing slow or blocking operations (like network or disk I/O) inside the predicate.
-   **Ignoring Concurrency of Context:** Do not pass a non-thread-safe context object or use a non-thread-safe predicate from multiple threads. While BlockDiamondUtil is safe, your callback may introduce race conditions if it modifies shared data without proper locking.

## Data Pipeline

BlockDiamondUtil is not part of a data persistence or network pipeline. It is a pure computational component. Its logical flow is self-contained.

> Flow:
> Calling System (e.g., Spell Engine) -> **BlockDiamondUtil.forEachBlock** -> (Invokes User Predicate for each block) -> World State Mutation or Query Result

