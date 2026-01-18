---
description: Architectural reference for TilledSoilBlock
---

# TilledSoilBlock

**Package:** com.hypixel.hytale.builtin.adventure.farming.states
**Type:** Data Component

## Definition
```java
// Signature
public class TilledSoilBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The TilledSoilBlock class is a data component within Hytale's Entity-Component-System (ECS) architecture. It does not represent a physical block in the world, but rather encapsulates the dynamic *state* of a block that has been tilled for farming. It is attached to a block entity managed by a ChunkStore, effectively augmenting a standard block like dirt with farming-specific properties.

Its primary architectural responsibilities are:
1.  **State Aggregation:** To serve as a plain data object holding all state related to a tilled soil block, such as its hydration level, fertilization status, and decay schedule.
2.  **Visual State Computation:** To provide logic via the computeBlockType method that translates its internal state into a specific block variant key. This key is used by the rendering and game logic engines to determine the block's appearance and behavior (e.g., `Fertilized_Watered`).
3.  **Data Persistence:** The static CODEC field is a critical part of its design. It defines a versioned serialization contract using the Hytale Codec system. This allows the component's state to be reliably saved to disk and loaded across different game versions, ensuring world compatibility and data integrity.

This component is a fundamental building block of the farming system, acting as the single source of truth for the condition of a single farm plot.

## Lifecycle & Ownership
-   **Creation:** An instance of TilledSoilBlock is created by server-side game logic, typically within the FarmingPlugin, when a player successfully uses a hoe on a compatible block. The newly created component is then attached to the target block's data container within the appropriate ChunkStore. It is also instantiated by the serialization system via its CODEC when a chunk is loaded from disk.
-   **Scope:** The component's lifetime is strictly bound to the state of the block it is attached to. It persists across server restarts as long as the block remains in a tilled state.
-   **Destruction:** The component is marked for garbage collection when its associated block reverts to a non-tilled state. This can be triggered by the `decayTime` elapsing, a player breaking the block, or another world event that destroys the farmland. The owning system is responsible for detaching the component from the ChunkStore.

## Internal State & Concurrency
-   **State:** The TilledSoilBlock is a highly mutable state container. Its fields are frequently modified by various game systems to reflect changes like watering, fertilizing, planting, and the passage of time. It holds no references to other systems and acts as a pure data object.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be owned and manipulated exclusively by the main server thread responsible for the game tick in its region. Any modification or access from asynchronous tasks or other threads must be marshaled back to the appropriate game thread to prevent race conditions, data corruption, and world state desynchronization.

## API Surface
The public API is designed for state manipulation and retrieval by trusted game systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, registered type identifier for this component. |
| computeBlockType(gameTime, type) | String | O(1) | Computes the block state key (e.g., "Watered") based on internal state. This is the primary logic method. |
| setPlanted(planted) | void | O(1) | Sets whether a crop is currently planted in this soil. |
| setWateredUntil(instant) | void | O(1) | Updates the timestamp until which the block is considered watered. |
| setFertilized(fertilized) | void | O(1) | Sets the fertilization status. |
| setDecayTime(instant) | void | O(1) | Sets the timestamp at which this tilled soil will revert to dirt if not maintained. |
| clone() | Component | O(1) | Creates a new TilledSoilBlock instance with an identical copy of the state. |

## Integration Patterns

### Standard Usage
Game logic should always retrieve the component from a ChunkStore for a specific block position, modify it, and then signal that the chunk data has changed.

```java
// Example: A system for watering a block
void waterBlock(World world, BlockPosition pos) {
    ChunkStore chunkStore = world.getChunkStoreFor(pos);
    ComponentType<ChunkStore, TilledSoilBlock> type = TilledSoilBlock.getComponentType();

    TilledSoilBlock soilState = chunkStore.getComponent(pos, type);

    if (soilState != null) {
        Instant now = world.getGameTime();
        Instant wateredUntil = now.plus(Duration.ofMinutes(30));
        soilState.setWateredUntil(wateredUntil);

        // Critical: Notify the chunk store of the modification for saving and replication.
        chunkStore.markDirty(pos);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation for Existing Blocks:** Do not use `new TilledSoilBlock()` to interact with a block that is already tilled. This creates a disconnected state object; you must fetch the existing instance from the ChunkStore to modify the actual world state.
-   **State Caching:** Do not hold a reference to a TilledSoilBlock instance across multiple game ticks. The component may be removed or replaced by other game logic at any time. Always re-fetch the component from the ChunkStore at the beginning of a transaction.
-   **Asynchronous Modification:** Never modify a TilledSoilBlock instance from a separate thread. This will bypass server authority and lead to severe state inconsistencies. Schedule a task on the main game thread to perform the modification.

## Data Pipeline
The TilledSoilBlock component is central to two primary data flows: persistence and real-time state updates.

**Persistence Pipeline (World Save/Load)**
> Flow:
> Chunk Unload -> World Save Trigger -> **TilledSoilBlock.CODEC** (Serialize) -> Binary Data -> Disk
>
> Chunk Load -> Disk -> Binary Data -> **TilledSoilBlock.CODEC** (Deserialize) -> **TilledSoilBlock** Instance -> Attached to Block in ChunkStore

**Game State Update Pipeline**
> Flow:
> Player Action (e.g., using a watering can) -> Server Game Event -> Farming System -> Modifies **TilledSoilBlock** state -> `computeBlockType` is called -> Block state is updated -> Update packet sent to Client -> Client renders new block texture

---

