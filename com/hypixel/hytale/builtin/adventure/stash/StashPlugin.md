---
description: Architectural reference for StashPlugin
---

# StashPlugin

**Package:** com.hypixel.hytale.builtin.adventure.stash
**Type:** Plugin / Service

## Definition
```java
// Signature
public class StashPlugin extends JavaPlugin {
```

## Architecture & Concepts

The StashPlugin is a server-side plugin responsible for the one-time population of item containers, such as chests or barrels, when they are first introduced into the world. It operates on a data-driven model where the contents of a container are determined by a *droplist* identifier stored on the container's component state.

This plugin's primary architectural contribution is the **StashSystem**, an ECS system that hooks into the world's entity lifecycle. When a chunk is loaded and an entity with an `ItemContainerState` component is added to the world, the StashSystem is triggered. It then invokes the core `stash` logic to populate the container based on its configured droplist.

This design decouples the act of populating a container from the container's creation. World designers can place thousands of containers and simply assign a droplist string to them. The StashPlugin ensures that this data is transformed into actual items at the appropriate moment—specifically, upon the container's first load in a non-Creative game mode. This prevents premature item generation and ensures loot is randomized (based on a deterministic seed) only when needed.

The plugin also registers a `StashGameplayConfig`, allowing server operators to configure behavior, such as whether the droplist should be cleared from a container after its contents have been generated.

## Lifecycle & Ownership

-   **Creation:** Instantiated once by the server's plugin loader during server startup. The constructor receives a `JavaPluginInit` context object which provides access to server registries.
-   **Scope:** The StashPlugin instance persists for the entire server session. The nested StashSystem is registered with the `ChunkStoreRegistry` and shares the same lifecycle.
-   **Destruction:** The plugin and its registered systems are discarded during server shutdown. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** The StashPlugin class is effectively stateless. It holds no mutable instance fields. Its primary role is to register the StashSystem during its `setup` phase. The static `stash` method operates exclusively on the arguments provided to it.
-   **Thread Safety:** The `setup` method is called from a single thread during server initialization. The core `stash` method is **not thread-safe** if called concurrently on the same `ItemContainerState` instance. However, its invocation by the StashSystem is safe, as the Hytale ECS framework guarantees that system methods like `onEntityAdded` are executed in a controlled, sequential manner for a given world update cycle.

    **WARNING:** Calling the static `stash` method from custom, multi-threaded code requires external locking on the `ItemContainerState` to prevent data corruption.

## API Surface

The primary public contract is the static utility method for populating a container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| stash(containerState, clearDropList) | ListTransaction | O(C) | Populates the given container from its droplist. C is the container's capacity. The complexity is dominated by shuffling the container's slots. Returns null if no droplist is defined. |

## Integration Patterns

### Standard Usage

This plugin is designed to work automatically. A developer or world designer enables its functionality by adding an `ItemContainerState` component to a block entity and setting its `droplist` property. The plugin handles the rest.

The following example illustrates the *effect* of the system, not direct invocation.

```java
// This is a conceptual example of what triggers the plugin.
// You do not write this code; the game engine does.

// 1. A chunk containing a chest is loaded.
// 2. The engine creates an entity and adds components.
ItemContainerState chestState = new ItemContainerState(...);
chestState.setDroplist("dungeon_common_loot"); // This is the key data point
chunkStoreEntity.addComponent(chestState);

// 3. The StashSystem.onEntityAdded is automatically invoked by the ECS,
//    which then calls StashPlugin.stash(chestState, ...).
//    The chest is now populated with items.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new StashPlugin()`. The server's plugin loader is solely responsible for creating and managing plugin instances.
-   **Manual Invocation:** Avoid calling `StashPlugin.stash()` directly unless you are managing the component lifecycle outside the standard world simulation. Doing so bypasses the game mode check (e.g., Creative mode) and the configured server settings.
-   **Ignoring Dependencies:** The internal StashSystem is configured to run *after* the `BlockStateModule.LegacyBlockStateRefSystem`. This ensures that the block state is fully initialized before the stash logic runs. Custom systems that interact with stashes should respect this dependency ordering.

## Data Pipeline

The flow of data from world definition to in-game items is a clear, event-driven pipeline orchestrated by the ECS.

> Flow:
> World Save Data -> Chunk Load -> Entity Creation -> **`ItemContainerState` Component Added** -> ECS triggers `StashSystem.onEntityAdded` -> **`StashPlugin.stash()`** -> `ItemModule` generates `ItemStack`s -> `ItemContainer` is populated -> (Optional) `droplist` field is cleared.

