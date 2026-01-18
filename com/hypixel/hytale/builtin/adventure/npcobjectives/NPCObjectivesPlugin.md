---
description: Architectural reference for NPCObjectivesPlugin
---

# NPCObjectivesPlugin

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives
**Type:** Singleton

## Definition
```java
// Signature
public class NPCObjectivesPlugin extends JavaPlugin {
```

## Architecture & Concepts

The NPCObjectivesPlugin serves as a critical bridge module, weaving together the server's core **Objective**, **NPC**, and **Entity Component System (ECS)** frameworks. As a `JavaPlugin`, it is automatically discovered and loaded by the server during its bootstrap sequence, extending the engine's base functionality with specialized features for NPC-driven quests.

Its primary architectural role is to register a suite of new game components and logic into their respective engine systems:

1.  **Objective Task Extension:** It introduces several new objective task types, such as `KillSpawnBeacon`, `Bounty`, and `KillNPC`, by registering them with the central `ObjectivePlugin`. This allows content creators to design quests that go beyond simple collection or interaction tasks.

2.  **NPC Behavior Integration:** It registers new core component types with the `NPCPlugin`, including `BuilderActionStartObjective` and `BuilderSensorHasTask`. These components directly expose objective-related logic to the NPC behavior tree system, enabling NPCs to dynamically assign quests, check player progress, and react to quest-related events.

3.  **ECS Logic Implementation:** The plugin registers ECS systems like `KillTrackerSystem` and resources like `KillTrackerResource`. This offloads the state management and continuous logic for tracking player kills to the highly performant ECS world, ensuring that quest progress is updated efficiently in response to game events.

4.  **Asset Loading Management:** By declaring explicit loading dependencies with `injectLoadsAfter`, the plugin guarantees that its dependent assets (like `ObjectiveAsset`) are parsed only after their own dependencies (`SpawnMarker`, `NPCGroup`) are fully loaded. This prevents race conditions and ensures data integrity during server startup.

In essence, this plugin is not a standalone system but an **integration hub** that enriches other core engine systems to create a cohesive and powerful NPC questing feature set.

### Lifecycle & Ownership

-   **Creation:** A single instance is created by the server's `PluginLoader` during the initial server startup phase. The constructor is invoked by the framework, which supplies a `JavaPluginInit` context object.
-   **Scope:** The plugin is a session-scoped singleton. The static `instance` field holds a reference that persists for the entire lifetime of the running server.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server is shut down and all plugins are unloaded.

## Internal State & Concurrency

-   **State:** The NPCObjectivesPlugin class itself is largely stateless. Its primary instance variable, `killTrackerResourceType`, is assigned once during the `setup` method and is effectively immutable thereafter. The plugin's main purpose is initialization and registration; it delegates all dynamic state management to the ECS systems and other plugins it configures.

-   **Thread Safety:** This class is **not thread-safe** and must be treated with extreme caution in a multi-threaded environment.
    -   The `setup` method is executed within the server's single-threaded initialization phase, making it inherently safe.
    -   The public static methods (`hasTask`, `updateTaskCompletion`, `startObjective`) are designed to be called from the main server game loop thread. They interact directly with non-thread-safe systems like the `ObjectiveDataStore` and the ECS `Store`.
    -   **WARNING:** Invoking these static methods from an asynchronous task or a different thread will lead to severe concurrency issues, including `ConcurrentModificationException`, data corruption, and server instability. All logic interacting with this plugin must be synchronized with the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | NPCObjectivesPlugin | O(1) | Returns the static singleton instance of the plugin. |
| hasTask(playerUUID, npcId, taskId) | boolean | O(1) avg | Checks if a player has an active objective task associated with a specific task ID. Delegates to ObjectiveDataStore. |
| updateTaskCompletion(store, ref, playerRef, npcId, taskId) | String | O(N*M) | Attempts to progress a `UseEntityObjectiveTask`. Returns the animation to play on success. Potentially slow if the player has many objectives (N) with many tasks (M). |
| startObjective(playerReference, taskId, store) | void | O(1) | Initiates a new objective for a player, identified by its task ID. Delegates to the core ObjectivePlugin. |
| getKillTrackerResourceType() | ResourceType | O(1) | Returns the handle for the KillTrackerResource, used by associated ECS systems. |

## Integration Patterns

### Standard Usage

Developers should not interact with this plugin's lifecycle. Instead, they should leverage the features it enables within other systems. The most common pattern is to use the registered NPC components within an NPC's behavior definition.

```java
// Example: A separate system or script triggering an objective for a player
// after a dialogue interaction.

// Assume 'playerRef' is a valid reference to a player entity
Ref<EntityStore> playerRef = ...;
Store<EntityStore> store = ...; // The current entity store

// Start the "escort_merchant" quest for the player
NPCObjectivesPlugin.startObjective(playerRef, "escort_merchant", store);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new NPCObjectivesPlugin()`. The server framework is solely responsible for its creation. Always use the static `NPCObjectivesPlugin.get()` to retrieve the singleton instance.
-   **Premature Access:** Do not call `NPCObjectivesPlugin.get()` from another plugin's constructor or early initialization phase. The static instance is only guaranteed to be non-null after the `setup` method has completed.
-   **Asynchronous Modification:** Never call `startObjective` or `updateTaskCompletion` from a separate thread. All modifications to the objective system must be performed on the main server thread to prevent race conditions.

## Data Pipeline

This plugin facilitates multiple data flows. A common example is the entire lifecycle of a "kill quest" given by an NPC.

> **Quest Assignment Flow:**
> Player interacts with NPC -> NPC Behavior Tree executes `BuilderActionStartObjective` node -> `NPCObjectivesPlugin.startObjective` is called -> `ObjectivePlugin` is invoked -> A new `Objective` with a `KillNPCObjectiveTask` is created -> State is saved in `ObjectiveDataStore`.

> **Quest Progress Flow:**
> Player kills a target entity -> A generic `KillEvent` is dispatched on the server -> The `KillTrackerSystem` (registered by this plugin) listens for this event -> The system updates the player's `KillTrackerResource` -> The system checks if the kill satisfies any active `KillNPCObjectiveTask` -> If so, it updates the task's completion count in the `ObjectiveDataStore`.

