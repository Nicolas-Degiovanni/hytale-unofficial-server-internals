---
description: Architectural reference for ReputationPlugin
---

# ReputationPlugin

**Package:** com.hypixel.hytale.builtin.adventure.reputation
**Type:** Singleton

## Definition
```java
// Signature
public class ReputationPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ReputationPlugin is the central authority for managing all reputation-related mechanics on the server. As a core `JavaPlugin`, it is loaded at server startup and provides a singleton service interface for other game systems to interact with reputation data.

Its primary architectural role is to bridge game logic with underlying data storage for reputation, which can be configured to be either player-specific or world-global. The plugin orchestrates several key engine systems:

*   **Asset System:** It registers and manages the lifecycle of `ReputationGroup` and `ReputationRank` assets. These assets are the data backbone of the system, defining the factions NPCs belong to and the reputation tiers players can achieve.
*   **Entity Component System (ECS):** It introduces the `ReputationGroupComponent`, which attaches a reputation group identity to an entity (typically an NPC). It also registers the `ReputationDataResource` for storing world-level reputation scores.
*   **Gameplay Configuration:** The plugin's behavior is dynamically controlled by the `ReputationGameplayConfig`. This configuration determines the fundamental storage strategy:
    *   **PerPlayer:** Each player has their own reputation map, tracking their standing with each group independently.
    *   **PerWorld:** A single, global reputation map is shared by all players in the world.
*   **Command & Scripting Interface:** By registering the `ReputationCommand`, it exposes administrative controls. It also registers `ReputationRequirement` with the dialogue choice system, allowing content creators to gate conversations or quests based on a player's reputation.

The core conceptual model involves a player entity interacting with an NPC entity. The NPC's `ReputationGroupComponent` identifies its faction. Actions that modify reputation call into this plugin, which then calculates the new score, clamps it within the defined min/max bounds, and persists it to the appropriate data store (either the player's `PlayerConfigData` or the world's `ReputationDataResource`).

### Lifecycle & Ownership
- **Creation:** The `ReputationPlugin` is instantiated once by the server's plugin loader during the bootstrap sequence. The static `instance` field is set within the `setup` method, establishing the singleton pattern.
- **Scope:** The plugin instance is global and persists for the entire lifetime of the server process. Its internal state, such as the cached list of `reputationRanks`, is initialized during the `start` phase and remains available until shutdown.
- **Destruction:** The object is eligible for garbage collection only when the server unloads all plugins during a shutdown procedure.

## Internal State & Concurrency
- **State:** The plugin maintains a significant amount of cached, mutable state.
    - **reputationRanks:** A list of all `ReputationRank` assets, loaded from disk and sorted during the `start` phase. This list is read frequently to determine a player's rank from their numerical score.
    - **minReputationValue / maxReputationValue:** Cached integers representing the absolute floor and ceiling for any reputation score, derived from the loaded `reputationRanks`.
    - **Component and Resource Types:** Handles to the ECS types (`reputationGroupComponentType`, `reputationDataResourceType`) are stored after registration. These are immutable references.

- **Thread Safety:** This class is **not** inherently thread-safe. It is designed to be operated within the server's main game loop or by systems that provide their own synchronization guarantees (e.g., world ticks).
    - The `setup` and `start` methods are executed in a single-threaded context during server initialization, making state setup safe.
    - Public methods like `changeReputation` and `getReputationValue` do not contain internal locks. They operate on data structures like `Object2IntMap` which are not thread-safe.
    - **WARNING:** Calling these methods from asynchronous tasks or parallel threads without external synchronization managed by the caller will lead to race conditions, data corruption, and server instability. The engine's `ComponentAccessor` and `Store` objects, passed as arguments, are assumed to provide the necessary transactional or thread-local context.

## API Surface
The public API provides a comprehensive interface for managing and querying reputation data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ReputationPlugin | O(1) | Static accessor for the singleton instance. |
| changeReputation(player, npcRef, value, accessor) | int | O(1) | Modifies a player's reputation with an NPC's group. The primary mutation entry point. |
| changeReputation(player, groupId, value, accessor) | int | O(1) | Modifies a player's reputation with a specific group ID. |
| changeReputation(world, groupId, value) | int | O(1) | Modifies the world's global reputation with a specific group ID. |
| getReputationValue(store, playerRef, npcRef) | int | O(1) | Retrieves the reputation value between a player and an NPC. |
| getReputationRank(store, playerRef, npcRef) | ReputationRank | O(N) | Retrieves the `ReputationRank` for a player relative to an NPC. N is the number of ranks. |
| getAttitude(store, playerRef, npc) | Attitude | O(N) | Retrieves the resulting `Attitude` an NPC should have towards a player based on reputation. |

## Integration Patterns

### Standard Usage
The plugin should always be accessed via its static `get()` method. The most common use case is to modify a player's reputation in response to a game event, such as completing a quest or choosing a dialogue option.

```java
// How a developer should normally use this
// Assume 'player' and 'npcRef' are available in the current context.
// 'accessor' is provided by the game engine's update logic.

ReputationPlugin plugin = ReputationPlugin.get();
if (plugin != null) {
    // Grant the player 50 reputation points with the NPC's group
    int newReputation = plugin.changeReputation(player, npcRef, 50, accessor);

    // Check the player's new rank
    ReputationRank rank = plugin.getReputationRank(accessor.getStore(), player.getEntityRef(), npcRef);
    if (rank != null) {
        // Trigger gameplay logic based on the new rank
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ReputationPlugin()`. The server's plugin loader is solely responsible for its creation. Attempting to create a second instance will break the singleton pattern and lead to unpredictable behavior.
- **Pre-Initialization Access:** Do not call methods that rely on asset data (like `getReputationRankFromValue`) from another plugin's `setup` method. The `reputationRanks` list is only populated during the `start` lifecycle phase, which runs after all plugins have been set up.
- **Direct Data Manipulation:** Avoid fetching the underlying reputation maps from `PlayerConfigData` or `ReputationDataResource` and modifying them directly. The `changeReputation` methods contain critical clamping logic that ensures values remain within the valid range defined by the `ReputationRank` assets. Bypassing this API can lead to data corruption.

## Data Pipeline
The flow of data for a reputation change is determined by the configured storage type.

**PerPlayer Storage Type:**
> Flow:
> Gameplay Event -> `ReputationPlugin.changeReputation(player, ...)` -> Reads `PlayerConfigData` -> Creates a copy of the player's reputation map -> `computeReputation()` -> Clamps value -> Writes updated map back to `PlayerConfigData`.

**PerWorld Storage Type:**
> Flow:
> Gameplay Event -> `ReputationPlugin.changeReputation(world, ...)` -> Reads `ReputationDataResource` from world entity store -> `computeReputation()` -> Modifies the resource's reputation map in-place -> Clamps value.

