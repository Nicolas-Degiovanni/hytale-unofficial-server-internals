---
description: Architectural reference for BenchState
---

# BenchState

**Package:** com.hypixel.hytale.builtin.crafting.state
**Type:** State Object

## Definition
```java
// Signature
public class BenchState extends BlockState implements DestroyableBlockState {
```

## Architecture & Concepts

The BenchState class is a specialized implementation of BlockState that encapsulates the dynamic, mutable data for a single crafting bench block within the game world. It serves as the critical link between a physical block's presence in a chunk, its static configuration defined in game assets, and the live state resulting from player interaction.

While a BlockType asset defines the immutable properties of all benches of a certain kind (e.g., upgrade recipes, tier names), a BenchState instance holds the data for one specific bench, such as its current `tierLevel` and any `upgradeItems` that have been partially applied.

Its primary architectural function is to be managed by the world's chunk system. The class is designed for persistence, featuring a static **CODEC** field. This codec dictates how an instance's state is serialized to disk when its parent chunk is unloaded and deserialized back into a live object when the chunk is loaded. This mechanism ensures that a bench's progress and tier level are preserved across game sessions.

## Lifecycle & Ownership

-   **Creation:** A BenchState instance is created by the world engine under two conditions:
    1.  **Deserialization:** When a chunk is loaded from disk, its data is passed through the BenchState CODEC, which constructs a new instance and populates it with the saved `tierLevel` and `upgradeItems`.
    2.  **Placement:** When a player places a new block that is configured to use this state model, the engine instantiates a default BenchState.

-   **Scope:** The object's lifetime is strictly tied to the existence of its corresponding block in the world. It persists as long as the block remains at its designated coordinates.

-   **Destruction:** The instance is marked for garbage collection when its associated block is destroyed (e.g., mined by a player). The engine invokes the `onDestroy` method immediately before destruction, providing a hook for critical cleanup logic. In this case, it triggers `dropUpgradeItems` to ensure any invested materials are not lost.

## Internal State & Concurrency

-   **State:** The internal state is highly mutable. The `tierLevel` and the `upgradeItems` array are frequently modified by player actions. Upon initialization, it also caches a reference to its corresponding `Bench` asset configuration for efficient access to upgrade rules. Any method that modifies persistent state, such as `addUpgradeItems` or `setTierLevel`, is responsible for calling `markNeedsSave` to flag the parent chunk for serialization.

-   **Thread Safety:** **This class is not thread-safe.** All interactions with a BenchState instance must be marshaled to the primary world thread. The `dropUpgradeItems` method demonstrates this pattern by using `world.execute` to schedule entity creation on the correct thread. Unsynchronized access from worker threads (e.g., networking or physics threads) will lead to race conditions, data corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialize(BlockType type) | boolean | O(1) | Binds the state to its static asset configuration. Called by the engine post-instantiation. |
| addUpgradeItems(List items) | void | O(N) | Appends items to the internal upgrade buffer. Marks the state for saving. |
| setTierLevel(int level) | void | O(1) | Sets the new tier level, triggers visual block updates, and marks the state for saving. |
| getNextLevelUpgradeMaterials() | BenchUpgradeRequirement | O(1) | Retrieves the recipe for the next tier from the cached Bench asset. |
| onDestroy() | void | O(M) | Lifecycle hook called by the engine. Ejects all stored upgrade items into the world as entities. |

## Integration Patterns

### Standard Usage

Logic interacting with a crafting bench should never instantiate BenchState directly. Instead, it must retrieve the state object from the world system using the block's position, then safely cast it to BenchState to access its specialized API.

```java
// A block interaction handler retrieves the state for a given position
BlockState state = world.getBlockState(blockPosition);

if (state instanceof BenchState) {
    BenchState bench = (BenchState) state;

    // Check if the provided items can be used for an upgrade
    BenchUpgradeRequirement req = bench.getNextLevelUpgradeMaterials();
    if (req.matches(playerInventory)) {
        // Consume items and apply them
        List<ItemStack> consumed = playerInventory.consume(req.getItems());
        bench.addUpgradeItems(consumed);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BenchState()`. The lifecycle is exclusively managed by the chunk and block state systems to ensure proper registration, persistence, and cleanup. Direct instantiation creates an orphaned object that will not be saved or updated correctly.

-   **State Caching:** Do not hold a reference to a BenchState instance for longer than the immediate scope of an operation. The block could be destroyed at any time, invalidating the cached reference and leading to errors. Always re-fetch the state from the world when you need to interact with it.

-   **Asynchronous Modification:** Do not modify a BenchState from a separate thread. All calls that change its state (e.g., `setTierLevel`) must be executed on the main world thread.

## Data Pipeline

The BenchState acts as a stateful processor for data flowing from player input to world persistence and rendering.

> **Flow 1: State Persistence**
> Player Interaction -> Block Interaction Handler -> `world.getBlockState(pos)` -> **BenchState**.`addUpgradeItems()` -> `markNeedsSave()` -> Chunk Serialization System -> Disk

> **Flow 2: Visual State Update**
> Upgrade Logic -> **BenchState**.`setTierLevel()` -> `onTierLevelChange()` -> `chunk.setBlockInteractionState()` -> World Renderer Update

> **Flow 3: Destruction & Cleanup**
> Player breaks block -> World Engine -> `onDestroy()` on **BenchState** -> `dropUpgradeItems()` -> Entity Management System -> New Item Entities in World

