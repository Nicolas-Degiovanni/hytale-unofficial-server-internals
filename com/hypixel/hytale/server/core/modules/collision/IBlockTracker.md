---
description: Architectural reference for IBlockTracker
---

# IBlockTracker

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IBlockTracker {
```

## Architecture & Concepts
The IBlockTracker interface defines a formal contract for systems that require efficient, persistent monitoring of a discrete set of block coordinates within the game world. It serves as a high-performance lookup mechanism, abstracting the underlying data structure used to store and query block positions.

This contract is fundamental for performance-sensitive server modules that cannot afford to iterate over large world regions. Key consumers include physics engines (for tracking collidable blocks), AI pathfinding (for monitoring dynamic obstacles), and specialized block entity systems (for ticking only active blocks).

By defining a common interface, the engine decouples various game systems from the specific implementation of the tracking mechanism. This allows the underlying storage strategy (e.g., HashSet, Octree, spatial hash grid) to be optimized or swapped without affecting the consumer modules.

## Lifecycle & Ownership
As an interface, IBlockTracker itself has no lifecycle. The following describes the expected lifecycle of its concrete implementations.

- **Creation:** Implementations are typically instantiated and managed by a parent "owner" system, such as a World instance or a dedicated CollisionModule. They are created when the parent system initializes.
- **Scope:** The lifetime of an IBlockTracker implementation is strictly tied to its owner. For example, a tracker associated with a specific world will exist for the duration of that world's session.
- **Destruction:** The object is eligible for garbage collection when its owning system is destroyed and all references to it are released. There is no explicit `destroy` or `close` method in the contract, so implementations must not hold unmanaged resources.

## Internal State & Concurrency
The IBlockTracker contract is inherently stateful. Implementations are expected to maintain an internal collection of tracked block positions.

- **State:** The internal state is mutable, modified by methods like `track`, `trackNew`, and `untrack`. The core responsibility of any implementation is the management of this collection of `Vector3i` coordinates.
- **Thread Safety:** **WARNING:** The interface provides no thread-safety guarantees. Implementations are not assumed to be thread-safe by default. Consumers must ensure that all interactions with a given IBlockTracker instance are synchronized externally or occur on a single, designated thread (e.g., the main server tick thread). Concurrent modification will lead to undefined behavior and state corruption.

## API Surface
The public API defines the complete set of operations for managing and querying a collection of block positions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPosition(int index) | Vector3i | O(1) / O(N) | Retrieves the position at a given index. Complexity depends heavily on implementation (e.g., Array vs. Set). |
| getCount() | int | O(1) | Returns the total number of currently tracked blocks. |
| track(int x, int y, int z) | boolean | O(1) | Adds the specified block coordinate to the tracked set if not already present. Returns true if the block was newly added. |
| trackNew(int x, int y, int z) | void | O(1) | Unconditionally adds the specified block coordinate. Assumes the block is not already tracked. Potentially faster than `track`. |
| isTracked(int x, int y, int z) | boolean | O(1) | Performs a high-performance check to determine if the specified coordinate is in the tracked set. |
| untrack(int x, int y, int z) | void | O(1) | Removes the specified block coordinate from the tracked set. Fails silently if the block is not present. |

**Note on Complexity:** The complexities listed assume a highly optimized, hash-based implementation. Alternative implementations (e.g., a simple list) would result in significantly worse performance for lookup and modification operations.

## Integration Patterns

### Standard Usage
A consumer system obtains a reference to an IBlockTracker implementation from a central service or world context. It then uses this shared instance to register blocks of interest or query the state of blocks relevant to its logic.

```java
// A system (e.g., physics) obtains the tracker from a world context
IBlockTracker collisionTracker = world.getCollisionModule().getBlockTracker();

// Registering a block that has become solid
collisionTracker.track(100, 64, -50);

// Querying if a block is relevant for collision checks
if (collisionTracker.isTracked(player.getX(), player.getY() - 1, player.getZ())) {
    // Perform ground collision logic
}
```

### Anti-Patterns (Do NOT do this)
- **Local Implementation:** Do not create a private implementation of IBlockTracker within a consumer class. This defeats the purpose of a shared, centralized tracking system and leads to data fragmentation and performance degradation.
- **State Assumption:** Do not call `getPosition(index)` and assume the order is stable between modifications. The internal ordering is not guaranteed by the contract.
- **Redundant Tracking:** Avoid calling `track` repeatedly on the same block within a single logic frame. Use `isTracked` to check for existence first if the state is unknown.

## Data Pipeline
IBlockTracker does not process a continuous stream of data. Instead, it functions as a stateful repository that responds to discrete API calls. The flow is one of control and state modification, not data transformation.

> **Flow:**
> 1. **Event Source:** A world event occurs (e.g., Block Placement, Block Update, Entity Movement).
> 2. **Controller Logic:** A system like WorldTicker or BlockManager processes the event.
> 3. **State Update:** The controller calls `track(x,y,z)` or `untrack(x,y,z)` on the **IBlockTracker** instance to update its state.
> 4. **State Query:** A separate system (e.g., PhysicsEngine) calls `isTracked(x,y,z)` on the same **IBlockTracker** instance.
> 5. **Decision:** The PhysicsEngine uses the boolean result to alter its behavior (e.g., apply collision).

