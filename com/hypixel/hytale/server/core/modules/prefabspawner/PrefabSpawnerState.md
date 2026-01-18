---
description: Architectural reference for PrefabSpawnerState
---

# PrefabSpawnerState

**Package:** com.hypixel.hytale.server.core.modules.prefabspawner
**Type:** State Object

## Definition
```java
// Signature
public class PrefabSpawnerState extends BlockState {
```

## Architecture & Concepts
The PrefabSpawnerState class is a specialized data container that represents the persistent configuration of a "Prefab Spawner" block within the game world. It is not an active service or manager; rather, it is a passive state object whose primary role is to hold parameters that guide the world generation system.

This class extends BlockState, signaling its tight coupling to the lifecycle of a single block at a specific coordinate. Its core architectural feature is the static **CODEC** field. This Hytale-specific serialization contract defines how an instance of PrefabSpawnerState is encoded to and decoded from persistent storage, such as a world chunk file. This mechanism makes the block's configuration durable across server restarts.

Furthermore, the class contains nested UI logic (PrefabSpawnerSettingsPage), indicating that its state is intended to be directly manipulated by players in-game. This establishes a direct link between player input, server-side state, and the eventual behavior of the world generator.

## Lifecycle & Ownership
-   **Creation:** An instance of PrefabSpawnerState is created by the Hytale serialization framework when a world chunk containing a prefab spawner block is loaded from disk. The static CODEC is used to deserialize the saved data into a new object. It is also instantiated when a player or system places a new prefab spawner block in the world.
-   **Scope:** The object's lifetime is strictly bound to the block it represents. It exists in memory only as long as its parent chunk is loaded. It is a fine-grained object, with potentially thousands of instances existing for a large world.
-   **Destruction:** The object is eligible for garbage collection as soon as its parent chunk is unloaded from server memory. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The internal state is entirely mutable. All fields are designed to be configured at runtime, primarily through the associated in-game settings page. The state includes the target prefab path, world generation flags, and weighted probabilities for spawning.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal locks or synchronization primitives. All mutations and reads must be performed by a system that guarantees thread safety, typically the main server world thread.

    **Warning:** Unsynchronized access from other threads (e.g., network packet handlers, asynchronous tasks) will lead to data corruption, inconsistent world generation, and race conditions during the world saving process.

## API Surface
The public API consists of simple accessors. The primary interaction contract is not the methods, but the data fields themselves, which are manipulated by the world generator and the UI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| prefabPath | String | O(1) | Defines the asset path to the prefab or folder of prefabs to be spawned. |
| fitHeightmap | boolean | O(1) | Controls if the spawned prefab should vertically align with the terrain's heightmap. |
| inheritSeed | boolean | O(1) | Determines if the child prefab uses the same worldgen seed as its parent, affecting procedural generation consistency. |
| inheritHeightCondition | boolean | O(1) | Controls if the child prefab must adhere to the same height-based spawning rules as its parent. |
| prefabWeights | PrefabWeights | O(1) | A data structure holding the weighted probabilities for selecting a prefab when prefabPath points to a directory. |
| markNeedsSave() | void | O(1) | Inherited from BlockState. Flags the object's parent chunk as dirty, ensuring its state is written to disk during the next save cycle. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. Instead, the system modifies its state in response to UI events. The following illustrates the server-side event handling that mutates the state.

```java
// Inside PrefabSpawnerSettingsPage.handleDataEvent
public void handleDataEvent(
    @Nonnull Ref<EntityStore> ref, 
    @Nonnull Store<EntityStore> store, 
    @Nonnull PrefabSpawnerState.PrefabSpawnerSettingsPageEventData data
) {
    // The 'state' field is the PrefabSpawnerState instance being modified
    this.state.prefabPath = data.prefabPath;
    this.state.fitHeightmap = data.fitHeightmap;
    this.state.inheritSeed = data.inheritSeed;
    this.state.inheritHeightCondition = data.inheritHeightCondition;
    this.state.prefabWeights = PrefabWeights.parse(data.prefabWeights);

    // Critical step: Flag the state as modified so it will be saved to disk
    this.state.markNeedsSave();

    // Close the UI for the player
    Player playerComponent = store.getComponent(ref, Player.getComponentType());
    playerComponent.getPageManager().setPage(ref, store, Page.None);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrefabSpawnerState()` for in-world blocks. The world engine manages the lifecycle. Manual creation will result in a disconnected state object that is not persisted.
-   **State Caching:** Do not cache an instance of PrefabSpawnerState in another long-lived service. Its lifetime is tied to a chunk, and holding a reference to it can cause memory leaks by preventing the chunk from being unloaded.
-   **Asynchronous Modification:** Do not modify a PrefabSpawnerState instance from a separate thread without explicit synchronization with the main world thread. This will corrupt the world save data.

## Data Pipeline
PrefabSpawnerState sits at the intersection of world storage, player interaction, and world generation. It has two primary data flows.

**Flow 1: Player Configuration Update**
> Player opens UI -> `PrefabSpawnerSettingsPage` is built from existing state -> Player changes values and clicks "Save" -> `PrefabSpawnerSettingsPageEventData` packet sent to server -> `handleDataEvent` mutates the **PrefabSpawnerState** instance -> `markNeedsSave()` is called -> World Save System serializes the updated state to the chunk file.

**Flow 2: World Generation Consumption**
> World Generator processes a chunk -> Encounters a Prefab Spawner block -> Retrieves the attached **PrefabSpawnerState** instance -> Reads properties like `prefabPath` and `fitHeightmap` -> Uses this data to select, place, and generate the specified prefab in the world.

