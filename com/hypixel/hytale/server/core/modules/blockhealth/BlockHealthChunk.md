---
description: Architectural reference for BlockHealthChunk
---

# BlockHealthChunk

**Package:** com.hypixel.hytale.server.core.modules.blockhealth
**Type:** Component

## Definition
```java
// Signature
public class BlockHealthChunk implements Component<ChunkStore> {
```

## Architecture & Concepts
The BlockHealthChunk is a server-side, stateful component that manages the health and fragility status of all blocks within a single world chunk. It does not represent the chunk itself, but rather an optional data layer attached to a ChunkStore. If no blocks in a chunk have ever been damaged, this component will not exist for that chunk, providing a significant memory optimization.

This class serves as the canonical data model for block damage. Core game systems, such as player mining or explosive damage, interact with this component to modify block state. It acts as a critical bridge between the game simulation and other engine systems:

1.  **Persistence:** It is responsible for serializing its own state into a binary format for storage. This is managed via the static CODEC field, which is invoked by the ChunkStore when a chunk is saved to disk.
2.  **Networking:** Upon modification, it immediately triggers calls to the World's NotificationHandler. This handler is responsible for creating and dispatching UpdateBlockDamage packets to relevant clients, ensuring that players see visual feedback like block cracks in real-time.

A single BlockHealthChunk instance corresponds to a single ChunkStore and therefore a single chunk in the world.

### Lifecycle & Ownership
-   **Creation:** A BlockHealthChunk is never instantiated directly. It is created under two conditions:
    1.  **On-Demand:** When a block within a previously undamaged chunk receives damage for the first time, the owning ChunkStore instantiates a new BlockHealthChunk and attaches it.
    2.  **On-Load:** When a chunk is loaded from storage that contains saved block health data, the component is instantiated and hydrated via its deserialize method.

-   **Scope:** The object's lifetime is strictly tied to its parent ChunkStore. It persists in memory as long as the chunk is loaded and active in the world simulation.

-   **Destruction:** The component is eligible for garbage collection when its parent ChunkStore is unloaded from memory. If all damaged blocks within the chunk are fully repaired, the internal maps may become empty, but the component object itself will persist until the chunk is unloaded.

## Internal State & Concurrency
-   **State:** This component is highly mutable. Its core state is held in two maps: blockHealthMap and blockFragilityMap. These maps store the precise health and temporary fragility status for individual blocks, keyed by their local coordinates within the chunk. It also maintains a lastRepairGameTime timestamp for tracking regeneration mechanics.

-   **Thread Safety:** **WARNING:** This class is not thread-safe. Its internal collections, Object2ObjectOpenHashMap, are unsynchronized. All read and write operations on a BlockHealthChunk instance **must** be performed on the main thread of the World it belongs to. Unsynchronized access from other threads will lead to data corruption, server instability, and unpredictable race conditions.

## API Surface
The public API provides the primary verbs for manipulating block durability.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| damageBlock(uptime, world, block, health) | BlockHealth | O(1) avg | Reduces a block's health. Notifies clients of the change. Returns the new health state. |
| repairBlock(world, block, progress) | BlockHealth | O(1) avg | Increases a block's health. Notifies clients of the change. Returns the new health state. |
| removeBlock(world, block) | void | O(1) avg | Removes all health data for a block, effectively resetting it to full health. Notifies clients. |
| makeBlockFragile(location, duration) | void | O(1) avg | Marks a block as fragile for a specified duration. |
| isBlockFragile(block) | boolean | O(1) avg | Checks if a block is currently marked as fragile. |
| getBlockHealth(block) | float | O(1) avg | Retrieves the current health of a block, returning full health if no damage is recorded. |
| createBlockDamagePackets(list) | void | O(N) | Populates a list with packets to synchronize the entire chunk's damage state with a client. N is the number of damaged blocks. |

## Integration Patterns

### Standard Usage
Game logic should never hold a long-term reference to a BlockHealthChunk. It should be retrieved from the ChunkStore, operated on, and then released. The ChunkStore is the authoritative owner.

```java
// Example from a hypothetical block interaction system
void onPlayerMine(PlayerRef player, Vector3i globalBlockPos) {
    World world = player.getWorld();
    ChunkStore chunkStore = world.getChunkStoreForBlock(globalBlockPos);

    // Request the component from the store. It will be created if it doesn't exist.
    BlockHealthChunk healthChunk = chunkStore.getOrCreateComponent(BlockHealthChunk.class);

    // Operate on the component
    Vector3i localBlockPos = toChunkLocal(globalBlockPos);
    float damage = calculatePlayerDamage(player);
    healthChunk.damageBlock(world.getUptime(), world, localBlockPos, damage);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call new BlockHealthChunk(). The component's lifecycle is exclusively managed by the ChunkStore and the Hytale component system. Direct instantiation will result in a disconnected, untracked state object that will not be persisted or synchronized.

-   **Cross-Thread Modification:** Do not access a BlockHealthChunk from an asynchronous task or a different world's thread. All interactions must be marshaled back to the owning world's main game loop to prevent data corruption.

-   **State Caching:** Do not cache a reference to a BlockHealthChunk instance across multiple ticks. The chunk may unload at any time, invalidating the reference and leading to NullPointerExceptions or use-after-free errors. Always re-request it from the ChunkStore when needed.

## Data Pipeline
The BlockHealthChunk sits at the center of two critical data flows: real-time game state updates and long-term persistence.

**Real-Time Damage Flow:**
> Player Input -> Server Game Logic -> ChunkStore.getComponent -> **BlockHealthChunk.damageBlock()** -> World.NotificationHandler -> Network Packet -> Client Render

**Persistence Flow (Save/Load Cycle):**
> Chunk Unload Event -> ChunkStore.save() -> **BlockHealthChunk.serialize()** -> Disk/Database -> Chunk Load Event -> ChunkStore.load() -> **BlockHealthChunk.deserialize()** -> In-Memory State

