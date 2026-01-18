---
description: Architectural reference for CoopBlock
---

# CoopBlock

**Package:** com.hypixel.hytale.builtin.adventure.farming.states
**Type:** Data Component

## Definition
```java
// Signature
public class CoopBlock implements Component<ChunkStore> {
```

## Architecture & Concepts
The CoopBlock is a server-side data component that attaches persistent state to a block in the world, transforming it into a functional NPC habitat. It is a core element of the farming and animal husbandry systems, acting as the authoritative data model for a single coop instance.

This component is not a self-contained system; rather, it is a stateful object managed by a higher-level service, likely the FarmingPlugin. It encapsulates all data and logic related to a coop's population, inventory, and lifecycle. Its responsibilities include:

-   **Resident Management:** Tracking which NPCs are assigned to the coop, whether they are currently spawned in the world or stored abstractly.
-   **Produce Generation:** Calculating and creating item drops from resident NPCs based on game time and asset configuration.
-   **Inventory Storage:** Maintaining an internal ItemContainer to hold generated produce until a player collects it.
-   **Lifecycle Hooks:** Providing methods that are invoked by external systems in response to world events, such as the coop block being placed, broken, or a chunk being loaded.

The CoopBlock is fundamentally tied to the Hytale Component system, but at the chunk level rather than the entity level. It is designed to be serialized and deserialed with the chunk data it belongs to.

### Lifecycle & Ownership
-   **Creation:** A CoopBlock component is instantiated and attached to a ChunkStore when a player places a corresponding coop block item in the world. Its initial state, particularly the coopAssetId, is derived from the placed item's data.
-   **Scope:** The component's lifetime is strictly bound to the physical block in the world grid. It persists across server restarts and chunk load/unload cycles through serialization.
-   **Destruction:** The component is destroyed when its associated block is broken. The handleBlockBroken method is invoked as a final cleanup step to eject resident NPCs and drop the contents of its inventory into the world.

## Internal State & Concurrency
-   **State:** The CoopBlock is highly mutable. Its primary state consists of the coopAssetId (linking to static configuration), a list of CoopResident objects, and an ItemContainer for storage. This state is frequently modified by game logic, such as when residents produce goods, are added to the coop, or are spawned into the world. The nested CoopResident class is a critical data structure that stores the state of each individual animal, including a persistent reference to its entity if spawned.

-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. The engine's architecture guarantees that component logic is executed on a single, deterministic world thread corresponding to the component's geographical region. Any attempt to access or modify a CoopBlock instance from an asynchronous task or a different world thread will lead to state corruption, race conditions, and server instability.

## API Surface
The public API provides hooks for the farming system to manage the coop's state in response to game events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryPutResident(metadata, worldTime) | boolean | O(N) | Attempts to add a captured NPC to the coop. Fails if the coop is full or does not accept the NPC type. |
| tryPutWildResidentFromWild(...) | boolean | O(N) | Attempts to tame a wild NPC into the coop. Attaches a CoopResidentComponent to the entity. |
| generateProduceToInventory(worldTime) | void | O(N) | The primary logic tick. Iterates residents, calculates time-based produce, and adds items to the internal inventory. |
| gatherProduceFromInventory(playerInv) | void | O(M) | Transfers all items from the coop's inventory to a player's inventory. M is the number of item stacks. |
| ensureSpawnResidentsInWorld(...) | void | O(N) | Spawns resident NPCs into the world near the coop if they are not already present. |
| ensureNoResidentsInWorld(store) | void | O(N) | Marks all spawned resident NPCs for despawn, effectively recalling them to the coop. |
| handleBlockBroken(...) | void | O(N+M) | Final cleanup logic. Spawns all residents and drops all inventory items into the world. |
| shouldResidentsBeInCoop(worldTime) | boolean | O(1) | Checks game time against asset configuration to determine if residents should be outside or inside. |

## Integration Patterns

### Standard Usage
The CoopBlock is exclusively managed by server-side systems. A system responsible for farming logic will acquire a reference to the component from the world's ChunkStore at a specific block position and invoke its methods as part of a scheduled update tick or in response to an event.

```java
// A hypothetical system ticking a coop component
CoopBlock coop = chunkStore.getComponent(blockPos, CoopBlock.getComponentType());
if (coop == null) {
    return; // Not a coop block
}

// On a scheduled update, generate produce
coop.generateProduceToInventory(world.getTimeResource());

// When a player interacts, give them the items
if (player.interactedWith(blockPos)) {
    coop.gatherProduceFromInventory(player.getInventory());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create a CoopBlock using its constructor. The component's lifecycle is managed by the world and ChunkStore. Manual creation will result in a disconnected, non-persistent state object that the game will not recognize.
-   **External State Mutation:** Do not directly access and modify the internal residents list or itemContainer. Use the provided API methods like tryPutResident and gatherProduceFromInventory to ensure all associated logic is executed correctly.
-   **Cross-Thread Access:** As stated in the concurrency section, this component must only be accessed from its designated world thread. Scheduling asynchronous tasks that modify a CoopBlock is a critical error.

## Data Pipeline
The CoopBlock acts as a stateful processor in several key data flows. The two most common are produce generation and resident spawning.

**Produce Generation Flow:**
> Flow:
> World Time Update -> Farming System Tick -> Get **CoopBlock** from ChunkStore -> `generateProduceToInventory()` -> Internal ItemContainer is populated -> Player Interaction Event -> `gatherProduceFromInventory()` -> Player Inventory

**Resident Spawning Flow (on Chunk Load):**
> Flow:
> Chunk Load Event -> Farming System -> Get **CoopBlock** from ChunkStore -> `shouldResidentsBeInCoop()` check -> `ensureSpawnResidentsInWorld()` -> NPCPlugin `spawnEntity()` -> New NPCEntity created in EntityStore -> **CoopBlock** state updated with PersistentRef

