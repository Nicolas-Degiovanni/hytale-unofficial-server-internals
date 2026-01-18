---
description: Architectural reference for ObjectivePlugin
---

# ObjectivePlugin

**Package:** com.hypixel.hytale.builtin.adventure.objectives
**Type:** Singleton

## Definition
```java
// Signature
public class ObjectivePlugin extends JavaPlugin {
```

## Architecture & Concepts
The ObjectivePlugin is the central nervous system for the entire quest and objective framework within the Hytale server. As a core `JavaPlugin`, it is loaded at server startup and orchestrates all aspects of objective management, from asset registration to live game state persistence.

Its primary architectural role is to act as a bridge between static data definitions (assets) and dynamic, stateful game objects. It consumes `ObjectiveAsset`, `ObjectiveLineAsset`, and other configuration files to create and manage live `Objective` instances for players.

Key responsibilities include:
- **Asset and Type Registration:** During its `setup` phase, the plugin registers all necessary asset types, ECS components (`ObjectiveHistoryComponent`), ECS systems (`ReachLocationMarkerSystems`), and custom codecs for tasks, completions, and interactions. This makes the objective system's data structures known to the wider engine.
- **Lifecycle Management:** It is the sole authority for creating, starting, progressing, completing, and canceling objectives. Methods like `startObjective` and `objectiveCompleted` define the state transitions for any given quest.
- **Factory and Registry:** Through methods like `registerTask` and `registerCompletion`, it provides an extensibility point. It maintains internal maps (`taskGenerators`, `completionGenerators`) that function as factories, allowing for the dynamic creation of specific task and completion logic based on asset definitions.
- **Persistence Layer:** It owns and configures the `ObjectiveDataStore`, which is responsible for serializing active objective state to disk and caching it in memory. This ensures quest progress is not lost across server restarts or player sessions.
- **Client Communication:** The plugin is responsible for notifying clients of objective state changes. It constructs and sends network packets like `TrackOrUpdateObjective` and `UntrackObjective` to ensure the player's UI accurately reflects their current quests.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the server's `JavaPlugin` loader during the server bootstrap sequence. The static singleton `instance` is assigned within the `setup` method, making it globally accessible via `ObjectivePlugin.get()`.
- **Scope:** The ObjectivePlugin is a session-scoped singleton. It persists for the entire lifetime of the server process.
- **Destruction:** The `shutdown` method is invoked by the plugin loader when the server is shutting down. This triggers a final, blocking save of all active objective data to disk via `objectiveDataStore.saveToDiskAllObjectives()`, preventing data loss.

## Internal State & Concurrency
- **State:** The plugin's state is highly mutable and central to the objective system.
    - It maintains concurrent maps of task and completion generators, which are populated during setup but can be read from multiple threads.
    - It holds direct references to critical infrastructure like the `ObjectiveDataStore` and registered ECS `ComponentType`s.
    - A scheduled task is created in the `start` method to periodically save objective data, running on a background thread pool.

- **Thread Safety:** The class is designed to operate in a multi-threaded server environment and employs specific patterns to ensure safety.
    - **WARNING:** Direct method calls are not universally thread-safe. Operations that modify world state (e.g., updating a player's components) must be dispatched to the appropriate world's execution thread.
    - The plugin frequently uses the `world.execute(...)` pattern within its event handlers (`onPlayerDisconnect`, `storeObjectiveHistoryData`) to serialize state-mutating operations onto the correct game thread, preventing race conditions within the ECS.
    - The use of `ConcurrentHashMap` for its factory registries allows for safe registration and access from different threads during initialization.
    - The `ObjectiveDataStore` is expected to handle its own internal synchronization for caching and disk I/O operations.

## API Surface
The public API provides high-level control over the objective lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ObjectivePlugin | O(1) | Retrieves the static singleton instance of the plugin. |
| startObjective(...) | Objective | O(N) | Starts a new objective for a set of N players. Involves asset lookup, state creation, persistence, and network synchronization. |
| startObjectiveLine(...) | Objective | O(N) | Starts the first objective in a predefined sequence (an ObjectiveLine). |
| objectiveCompleted(...) | void | O(N) | Finalizes a completed objective, stores history, grants rewards, and potentially starts the next objective in a line. |
| cancelObjective(...) | void | O(N) | Aborts an active objective and cleans up its associated state and entities. |
| registerTask(...) | void | O(1) | Extensibility point to register a new type of objective task. |
| registerCompletion(...) | void | O(1) | Extensibility point to register a new type of objective completion action. |
| addPlayerToExistingObjective(...) | void | O(1) | Adds a player to an already running objective. |
| removePlayerFromExistingObjective(...) | void | O(1) | Removes a player from an objective, potentially unloading it if no players remain. |

## Integration Patterns

### Standard Usage
The plugin is typically used by other game systems (e.g., interaction handlers, command executors) to trigger objective state changes. The singleton accessor is the only supported entry point.

```java
// Starting an objective for a set of players after an interaction
ObjectivePlugin plugin = ObjectivePlugin.get();
Store<EntityStore> store = player.getReference().getStore();
Set<UUID> participants = Set.of(player.getUuid());
UUID worldUUID = store.getExternalData().getWorld().getUuid();

// The world.execute call is critical for thread safety
store.getExternalData().getWorld().execute(() -> {
    plugin.startObjective("tutorial_quest_1", participants, worldUUID, null, store);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ObjectivePlugin()`. The server's plugin loader is responsible for its creation. Always use `ObjectivePlugin.get()` to retrieve the instance.
- **Ignoring World Threads:** Calling methods like `startObjective` or `objectiveCompleted` from an arbitrary background thread without dispatching the call to the correct `World` thread via `world.execute()` will cause severe concurrency issues, data corruption, and server crashes.
- **Direct DataStore Manipulation:** Avoid accessing the `ObjectiveDataStore` directly to modify objective state. The plugin's API methods handle necessary caching, state transitions, and network updates that would be bypassed.

## Data Pipeline

The plugin sits at the center of several key data flows.

**Objective Start Flow:**
> Game Event (e.g., Player Interaction) -> `ObjectivePlugin.startObjective` -> `ObjectiveAsset` Lookup -> `Objective` Instance Created -> `ObjectiveDataStore` Caches Instance -> `PlayerConfigData` Updated -> `TrackOrUpdateObjective` Packet Sent -> Client UI Updates

**Persistence Flow:**
> Game Progress (e.g., Task Completion) -> `Objective` state is mutated -> `objective.markDirty()` -> Scheduled Executor triggers `saveToDiskAllObjectives` -> `ObjectiveDataStore` finds dirty objectives -> `Objective.CODEC` serializes state -> `DiskDataStoreProvider` writes to file system

**Asset Reload Flow:**
> Asset File Change -> Asset Hot-Reload System -> `LoadedAssetsEvent` Fired -> `ObjectivePlugin.onObjectiveAssetLoaded` -> Active `Objective` instances reload their asset reference -> Game behavior updates live without a restart

