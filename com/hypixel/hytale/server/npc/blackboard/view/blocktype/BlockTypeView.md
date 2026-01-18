---
description: Architectural reference for BlockTypeView
---

# BlockTypeView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.blocktype
**Type:** Transient, Scoped Service

## Definition
```java
// Signature
public class BlockTypeView extends BlockRegionView<BlockTypeView> {
```

## Architecture & Concepts

The BlockTypeView is a fundamental component of the server-side NPC AI system, specifically within the Blackboard architecture. It functions as a spatially-partitioned query and caching layer, designed to efficiently answer the question: "Where is the nearest block of type X for an NPC?".

Each BlockTypeView instance represents a fixed, rectangular region of the world. Its primary architectural purpose is to perform **interest aggregation**. Instead of every NPC independently scanning the world for blocks they need, they register their "interest" (a list of desired block sets) with the BlockTypeView corresponding to their current location.

This design amortizes the high cost of chunk iteration and block scanning. The view aggregates the interests of all registered NPCs within its bounds and performs a single, optimized scan for all requested block types. The results of this scan are cached in a BlockPositionProvider component, which is attached directly to the world's chunk sections. This makes subsequent queries from any NPC in the same region for the same block types extremely fast, as they can read directly from the cache.

The BlockTypeView is the bridge between an NPC's abstract goal (e.g., "find wood") and the concrete world data (the coordinates of a specific log block).

### Lifecycle & Ownership
-   **Creation:** A BlockTypeView is instantiated on-demand by the central Blackboard service. An instance is created the first time an NPC enters a spatial region that does not already have an active view. The Blackboard is the sole owner and manager of BlockTypeView instances.

-   **Scope:** The object's lifetime is tied directly to the presence of NPCs within its spatial bounds. It persists as long as at least one entity is registered with it.

-   **Destruction:** When the last registered NPC leaves the view's region (by moving or being removed), the Blackboard system will prune and garbage collect the now-unused BlockTypeView instance.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful. Its core state includes:
    -   A set of references to all entities currently within its spatial bounds.
    -   A reference-counted map (blockSetCounts) tracking how many entities are searching for each specific block set.
    -   An aggregated BitSet (allBlockSets) representing the union of all block sets being searched for by any entity in the view.
    -   A lazily-updated list (blockSetAggregate) containing the flattened, concrete block IDs derived from allBlockSets, used to accelerate chunk scanning.

-   **Thread Safety:** **This class is not thread-safe and must not be accessed from multiple threads.** It is designed to be operated exclusively by the main server thread as part of the game loop. Any concurrent modification to its internal collections, especially the reference counters and entity set, will result in data corruption, race conditions, and unpredictable server behavior. All interactions must be synchronized with the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| findBlock(blockSet, range, ...) | IBlockPositionData | O(N\*M) | The primary query method. Scans all chunks (N) within the given range, potentially iterating all blocks (M) if the cache is cold. Throws IllegalStateException if internal state is corrupted. |
| isOutdated(ref, store) | boolean | O(1) | Performs a cheap check to determine if an entity has moved outside the spatial bounds of this view. |
| getUpdatedView(ref, accessor) | BlockTypeView | O(1) | Retrieves the correct view for an entity's current position and orchestrates the migration of its block set interests from the old view to the new one. |
| addSearchedBlockSets(ref, entity, sets) | void | O(k) | Registers an entity with this view and increments the reference count for its k desired block sets. |
| removeSearchedBlockSets(ref, entity, sets) | void | O(k) | Unregisters an entity and decrements the reference count for its k block sets. Triggers cleanup if it's the last entity. |

## Integration Patterns

### Standard Usage

The BlockTypeView is not intended for direct use by most game logic. It is an internal service managed by the Blackboard. High-level AI behaviors should interact with the Blackboard, which then delegates to the appropriate BlockTypeView instance.

```java
// Within an NPC's behavior tree or update logic:
// 1. Access the Blackboard via the ComponentAccessor
Blackboard blackboard = componentAccessor.getResource(Blackboard.getResourceType());

// 2. The Blackboard locates the correct BlockTypeView for the entity
//    and calls findBlock internally. The caller does not need to know
//    which view instance is being used.
IBlockPositionData targetBlock = blackboard.findBlock(
    entityRef,
    componentAccessor,
    TARGET_BLOCK_SET_ID,
    SEARCH_RANGE,
    true // pick a random one
);

if (targetBlock != null) {
    // Pathfind to targetBlock.getX(), targetBlock.getY(), targetBlock.getZ()
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BlockTypeView()`. The Blackboard is responsible for the entire lifecycle of views. Manually creating a view will result in a "rogue" instance that is not registered with the system and will not function correctly.
-   **State Tampering:** Do not directly access and modify internal collections like `entities` or `blockSetCounts`. Use the provided `addSearchedBlockSets` and `removeSearchedBlockSets` methods to ensure reference counting and cache invalidation logic is respected.
-   **Cross-Thread Access:** Never call `findBlock` or any other method from an asynchronous task or worker thread. All interactions must be serialized on the main server thread.

## Data Pipeline

The `findBlock` method initiates a complex data flow that transforms an abstract AI goal into a concrete world coordinate.

> Flow:
> NPC AI Task -> Blackboard Service -> **BlockTypeView.findBlock** -> World.getChunkIfInMemory -> ChunkStore -> Check for cached BlockPositionProvider
>
> **If Cache Miss/Stale:**
> **BlockTypeView** -> rebuildBlockTypeAggregate -> BlockSection -> **BlockPositionEntryGenerator.generate** -> Create new BlockPositionProvider -> Store cache on ChunkSection Component
>
> **If Cache Hit:**
> BlockPositionProvider.findBlocks -> Filter with ResourceView (for reserved blocks) -> Select closest or random block -> Return IBlockPositionData -> NPC AI Task

