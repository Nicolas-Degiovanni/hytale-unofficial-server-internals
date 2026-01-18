---
description: Architectural reference for BlockPlaceUtils
---

# BlockPlaceUtils

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Utility

## Definition
```java
// Signature
public class BlockPlaceUtils {
```

## Architecture & Concepts
BlockPlaceUtils is a stateless utility class that serves as the central authority for all server-side block placement logic. It is not a component or a service, but rather a collection of static methods that encapsulate a complex, multi-system transaction: placing a block in the world.

This class acts as an orchestrator, coordinating operations across several core engine domains:
- **Entity Component System (ECS):** It dispatches a PlaceBlockEvent, allowing other systems to inspect, modify, or cancel the placement operation. It also reads player data like GameMode.
- **World & Chunk Management:** It directly manipulates chunk data by calling methods on WorldChunk and BlockSection to set block types and states.
- **Asset Management:** It resolves block type keys and prefab asset IDs into concrete BlockType and PrefabListAsset definitions to determine placement rules, hitboxes, and associated prefabs.
- **Inventory System:** It interacts with the player's Inventory and ItemContainer to consume the placed block item, enforcing rules based on game mode.

Its primary architectural function is to decouple the high-level *intent* to place a block (e.g., from a network packet handler) from the low-level, high-consequence implementation of modifying the world state.

## Lifecycle & Ownership
- **Creation:** As a class containing only static methods, BlockPlaceUtils is never instantiated. It is loaded by the Java ClassLoader at server startup and exists for the entire application lifetime.
- **Scope:** The class and its methods are globally accessible. The lifecycle of concern is not that of the class itself, but of the data references (e.g., Ref<EntityStore>, Ref<ChunkStore>) passed into its methods. These references must be valid and scoped to an active world context.
- **Destruction:** The class is unloaded when the server application shuts down. No manual cleanup is required.

## Internal State & Concurrency
- **State:** BlockPlaceUtils is entirely stateless and immutable. It contains no instance or static fields that change during runtime, only static final constants for logging and messaging. All state is passed in as method arguments.
- **Thread Safety:** The methods in this class are **not thread-safe** by themselves. They perform direct, unsynchronized modifications to world state objects like WorldChunk. It is imperative that all calls to this utility are made from the designated world thread for the target chunk. The engine's use of `world.execute(...)` for prefab placement is an example of the correct pattern for deferring work to the appropriate thread.

**WARNING:** Calling methods from this class on an arbitrary worker thread (e.g., a network I/O thread) without proper synchronization or delegation to the world thread will result in world corruption, chunk deadlocks, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| placeBlock(...) | void | O(N) | Orchestrates the entire block placement transaction. Complexity is dependent on prefab size or the number of filler blocks. Throws runtime exceptions on critical failures. |
| canPlaceBlock(BlockType, String) | boolean | O(1) | Performs a fast check to see if a given block type allows being replaced by another, based on its placement override settings. |

## Integration Patterns

### Standard Usage
The intended use is within a server system that handles player interactions. The system must first gather all necessary context from the ECS and world stores before invoking the static method.

```java
// Within an ECS System that has access to the world and player entity
// Assume 'context' provides access to all necessary stores and references

Ref<EntityStore> playerRef = ...;
Ref<ChunkStore> chunkRef = ...;
Vector3i targetPosition = ...;
ItemStack itemInHand = ...;

// Delegate the entire complex operation to the utility
BlockPlaceUtils.placeBlock(
    playerRef,
    itemInHand,
    itemInHand.getBlockKey(),
    playerInventory.getHotbar(),
    placementNormal,
    targetPosition,
    blockRotation,
    playerInventory,
    activeSlot,
    true, // removeItemInHand
    chunkRef,
    context.getChunkStoreAccessor(),
    context.getEntityStoreAccessor()
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new BlockPlaceUtils()`. This is a utility class with static methods only.
- **Asynchronous Execution:** Do not call `placeBlock` from a network thread or any thread other than the main world thread responsible for the target chunk. This will bypass engine-level thread safety mechanisms.
- **Ignoring Event Cancellation:** The `placeBlock` method internally fires a cancellable `PlaceBlockEvent`. Systems should not attempt to bypass this by implementing their own block placement logic, as this would break other gameplay features that rely on intercepting the event.
- **Partial Context:** Providing invalid or null references for required parameters like `chunkReference` or `entityStore` will result in assertion errors or NullPointerExceptions.

## Data Pipeline
The `placeBlock` method represents a significant data processing pipeline that transforms a player's request into a permanent world change.

> Flow:
> Player Input -> Network Packet -> Interaction System -> **BlockPlaceUtils.placeBlock**
> 1.  **Pre-flight Checks:** Validate Y-coordinate.
> 2.  **Event Dispatch:** Create and invoke `PlaceBlockEvent` on the ECS event bus.
> 3.  **Cancellation Check:** Halt if event was cancelled by another system.
> 4.  **Inventory Transaction:** Consume item from the player's inventory (for Adventure mode).
> 5.  **Asset Validation:** Validate the block type key against known assets.
> 6.  **Prefab Path (Conditional):** If the block asset specifies a prefab, delegate to `PrefabUtil` to load and paste the prefab into the world. The pipeline terminates here for this block.
> 7.  **Single Block Path:**
>     a. Check world and environment permissions (`isBlockPlacementAllowed`, `isBlockModificationAllowed`).
>     b. Test if the block can physically fit using `worldChunk.testPlaceBlock`.
>     c. **Break & Drop:** Destroy the existing block at the target location and its filler blocks, dropping items if necessary via `BlockHarvestUtils`.
>     d. **World State Mutation:** Call `worldChunk.placeBlock` to commit the new block data to the chunk.
>     e. **State Hydration:** Apply `BlockState` metadata from the source `ItemStack` to the newly placed block.
>     f. **Ownership Tagging:** Update `PlacedByBlockState` and attach `PlacedByInteractionComponent` to link the block to the player who placed it.
>     g. **Neighbor Update:** Trigger connected block logic via `ConnectedBlocksUtil` to update adjacent blocks (e.g., fences, glass panes).
> 8.  **Chunk Update Propagation:** The modified chunk is now "dirty" and will be queued for saving and propagation to clients.

