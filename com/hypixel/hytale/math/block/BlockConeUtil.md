---
description: Architectural reference for BlockConeUtil
---

# BlockConeUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockConeUtil {
```

## Architecture & Concepts

BlockConeUtil is a stateless, procedural geometry utility for generating voxel-based elliptical cones. It is a fundamental tool used in world generation, structure placement, and certain visual effect systems that require manipulation of blocks in a conical volume.

The core design principle is **Inversion of Control (IoC)**. Instead of calculating and returning a potentially massive collection of block positions, which would incur significant memory allocation overhead, this utility implements an internal iterator. It accepts a functional interface, a TriIntObjPredicate, which is executed for every block coordinate within the generated shape. This allows the calling system to define the action (e.g., setting a block type, checking for air, adding to a particle system) without BlockConeUtil needing any knowledge of the world state.

The algorithm works by iterating vertically (along the Y-axis) and calculating the dimensions of the elliptical cross-section at each level. The radii of the ellipse are linearly interpolated from the base to the apex of the cone. A notable implementation detail is the addition of a small constant (0.41F) to the radii during calculation. This is a common voxel-space correction factor used to produce more visually appealing, rounded shapes and to mitigate aliasing artifacts at the edges of the discrete block grid.

The utility provides variants for creating solid cones, hollow cones (with a specified thickness), and inverted cones (apex at the bottom).

## Lifecycle & Ownership

BlockConeUtil is a static utility class and therefore has no instances. It cannot be created, owned, or destroyed in the traditional object-oriented sense.

- **Creation:** Not applicable. The class is loaded into the JVM by the ClassLoader at runtime, typically upon first access.
- **Scope:** Application-level. Its static methods are globally accessible throughout the application's lifetime.
- **Destruction:** Not applicable. The class is unloaded when the JVM shuts down.

## Internal State & Concurrency

- **State:** **Stateless and Immutable.** BlockConeUtil holds no internal state. All calculations are performed exclusively on the arguments provided to its static methods. Each method call is a pure function whose output (the sequence of calls to the consumer) is entirely determined by its inputs.

- **Thread Safety:** **Thread-safe.** Due to its stateless nature, all methods in BlockConeUtil can be safely called by multiple threads concurrently without risk of race conditions or data corruption within the utility itself.

    **WARNING:** While the utility is thread-safe, the caller is responsible for ensuring that the provided TriIntObjPredicate consumer is also thread-safe. If the consumer modifies shared state (e.g., a non-concurrent collection or the game world from an asynchronous thread), the caller must implement appropriate synchronization.

## API Surface

The API consists of overloaded static methods for generating different cone variations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(origin..., consumer) | static void | O(rX * rZ * h) | Iterates over all blocks in a solid, upright elliptical cone. The consumer is invoked for each coordinate. Iteration stops if the consumer returns false. |
| forEachBlock(origin..., thickness, consumer) | static void | O(rX * rZ * h) | Iterates over blocks in a hollow elliptical cone with a given wall thickness. |
| forEachBlockInverted(origin..., consumer) | static void | O(rX * rZ * h) | Iterates over all blocks in a solid, inverted (upside-down) elliptical cone. |
| forEachBlockInverted(origin..., thickness, consumer) | static void | O(rX * rZ * h) | Iterates over blocks in a hollow, inverted elliptical cone. |

## Integration Patterns

### Standard Usage

The primary use case is to modify the game world during procedural generation or in response to a game event. The caller provides a lambda expression that encapsulates the world modification logic.

```java
// Generate a solid cone of stone blocks pointing upwards
World world = context.getWorld();
int originX = 100;
int originY = 64;
int originZ = 200;

// The consumer predicate sets a block and always returns true to continue iteration.
TriIntObjPredicate<World> placeStone = (x, y, z, w) -> {
    w.setBlock(x, y, z, Block.STONE);
    return true;
};

BlockConeUtil.forEachBlock(originX, originY, originZ, 10, 20, 10, world, placeStone);
```

### Anti-Patterns (Do NOT do this)

- **Attempting Instantiation:** Do not attempt to create an instance with `new BlockConeUtil()`. This is a static utility class and cannot be instantiated.
- **Long-Running Consumers:** The consumer predicate is executed synchronously on the calling thread. Performing blocking operations such as network requests, heavy file I/O, or complex pathfinding inside the consumer will freeze the caller. If the caller is the main game thread, this will result in catastrophic performance degradation.
- **Unsafe World Modification:** If calling from an asynchronous worker thread, ensure that the consumer's interaction with the game world is thread-safe. Direct calls to methods like World.setBlock from off-thread are typically unsafe and must be scheduled for execution on the main game thread.

## Data Pipeline

BlockConeUtil does not process a stream of data in a traditional pipeline. Instead, it acts as a generator in a control flow, where input parameters define a geometric volume, and the output is a series of function calls.

> Flow:
> World Generation System -> **BlockConeUtil.forEachBlock**(params, consumer) -> [Internal Voxel Iteration] -> consumer.test(x, y, z) -> World State API -> World State Update

