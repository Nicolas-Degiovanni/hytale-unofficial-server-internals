---
description: Architectural reference for ParkourPlugin
---

# ParkourPlugin

**Package:** com.hypixel.hytale.builtin.parkour
**Type:** Singleton

## Definition
```java
// Signature
public class ParkourPlugin extends JavaPlugin {
```

## Architecture & Concepts
The ParkourPlugin serves as the central authority and state manager for all parkour-related gameplay on the server. It is implemented as a standard `JavaPlugin`, the primary mechanism for extending server-side game logic and features.

Architecturally, this plugin is deeply integrated with the server's core Entity Component System (ECS). Its primary function is to define and manage a new game concept—the parkour checkpoint—by introducing a custom component and the systems that operate on it.

Key architectural responsibilities include:
*   **Component Registration:** It registers the `ParkourCheckpoint` component, allowing any entity in the world to be designated as a checkpoint.
*   **System Registration:** It registers the `ParkourCheckpointSystems`, which contain the logic for detecting when a player interacts with a checkpoint and updating the game state accordingly.
*   **State Management:** It acts as a singleton container for the global state of all parkour courses and player progress. This includes tracking each player's current checkpoint, their start time, and the mapping of all checkpoint entities.
*   **Asset Dependency:** The plugin depends on a specific model asset, `Objective_Location_Marker`, to visually represent checkpoints in the world. Failure to load this asset is a fatal error for the plugin.

This class is the bridge between the low-level ECS framework and the high-level parkour game mode logic.

### Lifecycle & Ownership
- **Creation:** The ParkourPlugin is instantiated once by the server's plugin loader during the server bootstrap sequence. The constructor receives a `JavaPluginInit` context object, which provides access to server registries.
- **Scope:** The instance is a global singleton that persists for the entire lifetime of the server process. Its state represents all parkour activity since the last server start.
- **Destruction:** The object is decommissioned and becomes eligible for garbage collection only when the server is shutting down or if the plugin is explicitly unloaded by an administrator.

## Internal State & Concurrency
- **State:** The ParkourPlugin is highly stateful and mutable. It maintains several maps that serve as the single source of truth for player progress and course layout:
    - `currentCheckpointByPlayerMap`: Tracks the current checkpoint index for each participating player.
    - `startTimeByPlayerMap`: Records the system time when a player started a parkour run.
    - `checkpointUUIDMap`: Maps a sequential checkpoint index to the unique entity UUID of the checkpoint.
    These collections are modified in real-time as players interact with the parkour course.

- **Thread Safety:** **WARNING:** This class is not thread-safe. The internal state is managed by non-concurrent collections (e.g., `Object2IntOpenHashMap`). All method calls that read or mutate plugin state **must** be executed on the main server thread. Asynchronous access from other threads without external locking will lead to race conditions, data corruption, and server instability.

## API Surface
The public API is primarily designed for consumption by the associated `ParkourCheckpointSystems` and `ParkourCommand`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static ParkourPlugin | O(1) | Retrieves the global singleton instance of the plugin. |
| resetPlayer(UUID playerUuid) | void | O(1) | Resets a specific player's parkour progress, setting their current checkpoint to the start. |
| updateLastIndex() | void | O(N) | Recalculates the highest checkpoint index. N is the number of registered checkpoints. |
| getParkourCheckpointComponentType() | ComponentType | O(1) | Returns the registered ECS component type for parkour checkpoints. |
| getCurrentCheckpointByPlayerMap() | Object2IntMap | O(1) | **Warning:** Returns a direct reference to the internal state map. External modification is an anti-pattern. |

## Integration Patterns

### Standard Usage
The plugin is designed to be accessed as a singleton from other server-side code, such as commands or ECS systems, to query or manipulate parkour state.

```java
// Typical usage within another system or command
ParkourPlugin plugin = ParkourPlugin.get();

if (plugin == null) {
    // This can happen if the plugin fails to load or is disabled.
    // Always null-check before use.
    return;
}

// Reset a player's progress after they finish or leave
plugin.resetPlayer(somePlayer.getUuid());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ParkourPlugin()`. The server's plugin loader is solely responsible for its lifecycle. Attempting to create a second instance will break the singleton pattern and lead to a completely broken game state.
- **Unsynchronized Access:** Do not access the plugin's methods or state maps from an asynchronous task or a different thread. This will cause critical race conditions. All logic must be scheduled to run on the main server thread.
- **External State Mutation:** Do not directly modify the maps returned by getters like `getCurrentCheckpointByPlayerMap()`. This bypasses the plugin's internal logic and can lead to an inconsistent and unpredictable state.

    ```java
    // DO NOT DO THIS
    // This is an unsupported and dangerous modification of internal state.
    ParkourPlugin.get().getCurrentCheckpointByPlayerMap().put(playerUuid, 100);
    ```

## Data Pipeline
The plugin's primary data flow is triggered by player movement and processed through the ECS.

> Flow:
> Player Movement Packet -> Server Physics Engine -> Entity Collision Event -> **ParkourCheckpointSystems.Ticking** -> Reads Player & Checkpoint Data -> Calls **ParkourPlugin.get().someMethod()** -> Mutates Internal State Maps -> (Optional) Network Packet to Client for UI Update

