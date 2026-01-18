---
description: Architectural reference for BlockAccessor
---

# BlockAccessor

**Package:** com.hypixel.hytale.server.core.universe.world.accessor
**Type:** Abstraction / Contract

## Definition
```java
// Signature
public interface BlockAccessor {
```

## Architecture & Concepts

The BlockAccessor interface is a foundational contract within the server architecture for all block-level world manipulation. It serves as a high-level, transactional API that abstracts the underlying physical storage of chunk data, which may be complex and highly optimized.

This interface is not intended to be implemented by game-logic systems. Instead, engine-level systems like the ChunkAccessor provide concrete implementations. BlockAccessor acts as a facade, exposing a clean and consistent set of operations (getBlock, setBlock, placeBlock) while hiding the intricate details of data serialization, palette mapping, and bit-packed block states.

An individual BlockAccessor instance is conceptually bound to a specific vertical column of chunks, identified by its X and Z coordinates. While it operates on world-space coordinates, its internal implementation is optimized for accesses within its designated chunk column. For operations that may cross chunk boundaries, such as validating the placement of a large structure via *testBlockTypes*, the accessor uses its parent ChunkAccessor to query adjacent chunk data.

The API surface is rich with default methods, providing convenient overloads for setting blocks using asset keys, raw integer IDs, or BlockType objects. This design promotes ease of use while the core, primitive operations remain minimal and performant.

## Lifecycle & Ownership

As an interface, BlockAccessor itself has no lifecycle. The lifecycle of its *implementations* is critical to understand.

-   **Creation:** Concrete BlockAccessor instances are created on-demand by the world system. They are typically retrieved for a specific, short-lived operation, for example: `world.getChunkAccessor(chunkX, chunkZ)`. They are never meant to be instantiated directly by gameplay code.
-   **Scope:** An accessor is a lightweight, temporary handle. Its lifespan should be confined to the scope of a single method or a single transaction. **WARNING:** Caching or storing a reference to a BlockAccessor is a severe anti-pattern. The underlying chunk it references may be unloaded by the world system at any time, leading to stale data, NullPointerExceptions, or world corruption.
-   **Destruction:** Instances are managed by the Java garbage collector. Once all references are dropped, the object is reclaimed. There is no explicit `close` or `destroy` method.

## Internal State & Concurrency

-   **State:** The interface itself is stateless. However, any concrete implementation is inherently a proxy to the **mutable** state of the game world. Operations on the accessor directly modify the underlying chunk data.
-   **Thread Safety:** **CRITICAL:** Implementations of BlockAccessor are **not thread-safe**. All world modification operations must be performed exclusively on the main server thread (the primary tick thread). Any attempt to read or write block data from an asynchronous task or a different thread without explicit, engine-provided synchronization mechanisms will result in catastrophic race conditions, data corruption, and server instability.

## API Surface

The following table highlights the primary methods of the BlockAccessor contract. Convenience overloads are omitted for clarity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlock(x, y, z) | int | O(1) | Retrieves the numerical ID of the block at the given world coordinates. |
| setBlock(...) | boolean | O(1) | Sets the block at the given coordinates. Returns false if the operation fails. |
| breakBlock(x, y, z) | boolean | O(1) | A specialized `setBlock` operation that replaces the target block with air. |
| placeBlock(...) | boolean | O(N) | A complex operation to place a block with specific rotation, performing validation checks against its hitbox. N is the number of sub-blocks in the hitbox. |
| testBlockTypes(...) | boolean | O(N) | Tests if a block and its associated hitbox can be placed at a location, based on a predicate. Does not modify the world. |
| setTicking(x, y, z, isTicking) | boolean | O(1) | Marks or unmarks a block to receive periodic game ticks for random updates. |

**Deprecation Notice:** Several methods related to direct state, filler, and fluid access are marked as deprecated. This indicates a shift in the engine's design towards a more unified block state model, likely consolidated into the `settings` integer parameter of the `setBlock` method. Avoid using any deprecated symbols.

## Integration Patterns

### Standard Usage

A BlockAccessor should be retrieved, used, and discarded within a single, atomic operation. The standard pattern involves getting the accessor from a world or entity context, performing the necessary block modifications, and then releasing the reference.

```java
// Correctly retrieve an accessor for a specific world location
BlockAccessor accessor = world.getChunkAccessorForBlock(targetPosition);

// Perform a transactional operation
if (accessor.getBlockType(targetPosition) == BlockType.GRASS) {
    accessor.setBlock(targetPosition.x, targetPosition.y, targetPosition.z, "hytale:stone");
}
// The 'accessor' reference should now go out of scope and be garbage collected.
```

### Anti-Patterns (Do NOT do this)

-   **Long-Lived References:** Do not store a BlockAccessor instance in a component field or as a static variable. The underlying chunk can be unloaded, invalidating the accessor.
    ```java
    // BAD: Storing the accessor in a component
    public class MyComponent {
        private BlockAccessor cachedAccessor; // This will become stale and cause bugs.
    }
    ```
-   **Cross-Thread Access:** Never pass a BlockAccessor to another thread or use it in a parallel stream. All access must be on the main server thread.
    ```java
    // BAD: Using the accessor in a new thread
    new Thread(() -> {
        // This will corrupt world state and crash the server.
        accessor.setBlock(0, 64, 0, "hytale:dirt");
    }).start();
    ```
-   **Ignoring Return Values:** Methods like `placeBlock` and `setBlock` return a boolean indicating success. Ignoring this can lead to logical errors where subsequent code assumes a block was placed when it was actually obstructed.

## Data Pipeline

The BlockAccessor is a central component in the data flow for world modification. It sits between high-level game logic and the low-level chunk storage system.

**Example Flow: Player Breaks a Block**
> Player Input -> Client Network Packet -> Server Network Handler -> Player Interaction System -> **BlockAccessor.breakBlock()** -> ChunkStore Data Update -> World Change Propagation System -> Server Network Packet (to all clients)

**Example Flow: AI Pathfinding Query**
> AI System (Pathfinder) -> World Query -> **BlockAccessor.getBlockType()** -> ChunkStore Data Read -> Return BlockType to AI System

