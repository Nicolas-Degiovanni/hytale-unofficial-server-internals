---
description: Architectural reference for FitsAPortal
---

# FitsAPortal

**Package:** com.hypixel.hytale.builtin.portals.utils.posqueries.predicates
**Type:** Utility

## Definition
```java
// Signature
public class FitsAPortal implements PositionPredicate {
```

## Architecture & Concepts
The FitsAPortal class is a concrete implementation of the PositionPredicate strategy interface. Its sole responsibility is to perform a fine-grained, multi-block validation to determine if a specific volume in the world can physically accommodate a portal structure.

Architecturally, it serves as a low-level world state query engine, encapsulating the precise geometric and material requirements for a valid portal placement. It operates directly on fundamental world data structures like ChunkStore, BlockChunk, and BlockSection. This direct access allows it to enforce strict rules about the environment, such as requiring a solid foundation and an empty volume for the portal frame, while also ensuring the area is free of fluids.

This predicate is designed to be used by higher-level systems, such as world generation services or player interaction handlers, which need to find or validate potential portal locations without needing to know the specific implementation details of the check.

## Lifecycle & Ownership
- **Creation:** FitsAPortal is a lightweight, stateless object. It is typically instantiated on-demand by a system that needs to perform a position validation. There is no central manager or factory for this class.
- **Scope:** The lifetime of a FitsAPortal instance is expected to be transient and confined to the scope of a single method call or operation. For example, an instance might be created, used as an argument to a search algorithm, and then immediately become eligible for garbage collection.
- **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. No manual destruction or resource release is necessary.

## Internal State & Concurrency
- **State:** This class is **stateless**. It holds no instance fields and its behavior is exclusively determined by the arguments passed to its methods. The internal THREES array is a static final constant and does not contribute to instance state.

- **Thread Safety:** The class itself is inherently thread-safe. However, the methods operate on the World object, which is a complex, mutable structure.

    **WARNING:** The underlying World and ChunkStore components are **not** thread-safe. All invocations of the test or check methods must be performed on the main server world-update thread to prevent catastrophic race conditions, such as reading partially loaded chunks or accessing block data while it is being modified by another system.

## API Surface
The public contract consists of the interface method and its static implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(World world, Vector3d point) | boolean | O(1) | Implements the PositionPredicate interface. Validates a 3x5x3 block volume centered on the point. This is a high-cost check with a large constant factor. |
| check(World world, Vector3d point) | boolean | O(1) | Static implementation of the core validation logic. Performs the same high-cost check as the test method. |

## Integration Patterns

### Standard Usage
FitsAPortal should be used as a predicate within a broader position-querying or world-modification system. It is passed to an algorithm that iterates through potential locations.

```java
// A hypothetical position searcher uses the predicate to find a valid spot.
PositionSearcher searcher = new PositionSearcher(world, player.getPosition());
Optional<Vector3d> validPortalLocation = searcher.findFirst(new FitsAPortal());

if (validPortalLocation.isPresent()) {
    // Proceed with portal creation at the validated location.
    world.createPortal(validPortalLocation.get());
}
```

### Anti-Patterns (Do NOT do this)
- **High-Frequency Polling:** Do not invoke this predicate in a tight loop or on every game tick. The check is computationally expensive, involving 45 separate block state lookups, chunk component access, and asset map queries. Use it for discrete, infrequent events like a player command or a single world-generation pass.
- **Asynchronous Execution:** Never call this method from an asynchronous task or a separate thread without explicit, robust synchronization with the main world thread. Doing so will lead to data corruption, crashes from ConcurrentModificationExceptions, or logically incorrect results based on stale world data.

## Data Pipeline
FitsAPortal does not process a stream of data; rather, it executes a complex query against the world state. The flow of this query is as follows:

> Flow:
> Calling System -> **FitsAPortal.test(world, point)** -> Iterates 3x5x3 Volume -> For each block: [World::getChunk -> ChunkStore::getComponent -> WorldUtil::getFluidIdAtPosition -> BlockChunk::getSectionAtBlockY -> BlockSection::get -> BlockType::getAssetMap] -> Final boolean Result

