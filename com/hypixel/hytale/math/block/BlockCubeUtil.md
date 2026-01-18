---
description: Architectural reference for BlockCubeUtil
---

# BlockCubeUtil

**Package:** com.hypixel.hytale.math.block
**Type:** Utility

## Definition
```java
// Signature
public class BlockCubeUtil {
```

## Architecture & Concepts

BlockCubeUtil is a foundational, stateless utility for performing high-performance, volumetric iterations over the world grid. It is not a managed service or a component within the main game loop; rather, it is a low-level toolset used by higher-level systems that need to query or manipulate regions of blocks.

The core design is an implementation of the **Visitor Pattern**. The various `forEachBlock` methods act as drivers that traverse a defined 3D space (a cuboid), while the caller-provided `TriIntObjPredicate` acts as the "visitor". This predicate is invoked for each block coordinate within the volume, allowing for generic, reusable iteration logic.

A key architectural feature is its **short-circuiting** behavior. The iteration is immediately terminated if the consumer predicate returns `false`. This makes the utility highly efficient for search operations, as the traversal can stop the moment a condition is met, avoiding unnecessary computation on the remaining volume.

This class is central to systems such as:
*   World generation and post-processing (e.g., placing features, smoothing terrain).
*   Player building and destruction mechanics.
*   AI environmental analysis (e.g., pathfinding, line-of-sight checks).
*   Large-scale physics simulations (e.g., checking for block updates in a region).

## Lifecycle & Ownership

As a static utility class, BlockCubeUtil does not have a traditional object lifecycle.

*   **Creation:** The class is never instantiated. Its bytecode is loaded into the JVM by the ClassLoader on first use.
*   **Scope:** Application-level. Its static methods are available globally throughout the application's runtime.
*   **Destruction:** The class is unloaded from memory only when the JVM shuts down. There is no manual cleanup or destruction process.

## Internal State & Concurrency

*   **State:** BlockCubeUtil is completely **stateless** and **immutable**. It maintains no internal state between method calls. All computations are performed exclusively on stack-allocated variables and the arguments passed to each method.

*   **Thread Safety:** The methods in this class are inherently **thread-safe**. However, this guarantee does not extend to the consumer predicate provided by the caller.

    **WARNING:** The caller is solely responsible for ensuring that the logic within the supplied `TriIntObjPredicate` is thread-safe if it accesses or modifies shared resources. BlockCubeUtil itself introduces no locks or synchronization primitives. Concurrent iteration using a non-thread-safe consumer will lead to race conditions and undefined behavior.

## API Surface

The public API consists of several overloaded static methods for iterating over block volumes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| forEachBlock(origin, radii, t, consumer) | boolean | O(W×H×D) | Iterates a solid cuboid. Returns false if the consumer short-circuits the loop. |
| forEachBlock(pointOne, pointTwo, t, consumer) | boolean | O(W×H×D) | Iterates a solid Axis-Aligned Bounding Box (AABB) defined by two corner points. |
| forEachBlock(..., thickness, ..., hollow, t, consumer) | boolean | O(W×H×D) | Iterates the shell of a cuboid. The `hollow` flag determines if the empty inner volume is also iterated. |

## Integration Patterns

### Standard Usage

The primary pattern is to invoke a static `forEachBlock` method, passing a lambda expression or method reference as the consumer. This is commonly used to search for a block or apply an effect to a region.

```java
// Example: Find the first "gold_ore" block within a 10-block radius of the player.
Vector3i playerPosition = player.getPosition();
Vector3i searchOrigin = new Vector3i(playerPosition);
final AtomicReference<Vector3i> foundPosition = new AtomicReference<>();

// The World object is passed as the generic type T for context.
BlockCubeUtil.forEachBlock(searchOrigin.x, searchOrigin.y, searchOrigin.z, 10, 10, 10, world, (x, y, z, w) -> {
    if (w.getBlockAt(x, y, z).isType("gold_ore")) {
        foundPosition.set(new Vector3i(x, y, z));
        return false; // Short-circuit: stop searching once found.
    }
    return true; // Continue searching.
});

if (foundPosition.get() != null) {
    // Gold ore was found at foundPosition.get()
}
```

### Anti-Patterns (Do NOT do this)

*   **Attempted Instantiation:** Do not call `new BlockCubeUtil()`. This class is not designed to be instantiated and provides no instance-level functionality.
*   **Large Synchronous Iterations:** Avoid calling `forEachBlock` on extremely large volumes (e.g., an entire chunk) from the main game thread. This will block rendering and cause severe client-side stutter. Offload large world-scanning operations to a dedicated worker thread pool.
*   **Unsafe Concurrent Modification:** Do not use a consumer that modifies a shared collection (e.g., an ArrayList) from multiple threads without proper synchronization. This is a classic concurrency bug that BlockCubeUtil will not prevent.

## Data Pipeline

BlockCubeUtil acts as an iterator or data generator, not a processing pipeline. Its role is to produce a stream of coordinates that feed into other systems.

> Flow:
> High-Level System (e.g., World Generator) -> **BlockCubeUtil.forEachBlock**(region) -> Generates (x, y, z) Coordinates -> Caller's Consumer Predicate -> World Data Access (e.g., `world.setBlockAt(x,y,z)`)

