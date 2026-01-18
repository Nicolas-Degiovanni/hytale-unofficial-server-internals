---
description: Architectural reference for NPCPlugin
---

# NPCPlugin

**Package:** com.hypixel.hytale.server.npc
**Type:** Singleton

## Definition
```java
// Signature
public class NPCPlugin extends JavaPlugin {
```

## Architecture & Concepts

The NPCPlugin is the central orchestration point for the entire server-side Non-Player Character (NPC) system. It functions as a foundational module within the server's plugin architecture, acting as the composition root for all NPC-related logic, data, and behavior. Its primary architectural role is to bridge the gap between data-driven NPC definitions (JSON asset files) and the live, in-game execution of NPC artificial intelligence within the Entity Component System (ECS).

This class is responsible for several critical domains:

*   **System & Component Registration:** During the server bootstrap phase, NPCPlugin registers a vast number of ECS systems and components. These systems implement the core AI logic, such as pathfinding (SteeringSystem), behavior execution (RoleSystems), state evaluation (StateEvaluatorSystem), and spatial awareness (NPCSpatialSystem).
*   **Behavior Factory Management:** It initializes and manages the **BuilderManager**, a sophisticated factory and registry system. The **BuilderManager** is responsible for parsing NPC role definitions from assets and constructing executable behavior objects (Sensors, Actions, BodyMotions). This data-driven design allows designers and modders to create complex AI without writing new Java code.
*   **Asset Lifecycle Management:** The plugin subscribes to asset loading and unloading events. This allows it to dynamically respond to changes in NPC configurations, such as role definitions, attitude groups, and balancing data, enabling hot-reloading of NPC behavior while the server is running.
*   **Facade for NPC Spawning:** It provides the primary, high-level API (**spawnNPC**) for creating new NPCs in the world. This method abstracts the underlying complexity of entity creation and initial component setup.
*   **Service Location:** As a singleton, it serves as a central service locator for other systems to access core NPC resources like the **AttitudeMap** or the **BuilderManager**.

## Lifecycle & Ownership

-   **Creation:** The NPCPlugin is instantiated once by the server's core plugin loader during the initial server startup sequence. Its constructor requires a **JavaPluginInit** context, which is only available during this bootstrap phase.
-   **Scope:** The NPCPlugin instance is a singleton that persists for the entire lifetime of the server process. The static **get** method provides global access to this single instance.
-   **Destruction:** The instance is destroyed and its resources are released only when the server shuts down or if the plugin is explicitly unloaded by the server's plugin manager.

## Internal State & Concurrency

-   **State:** The NPCPlugin maintains a significant amount of mutable state. This includes the **BuilderManager** which caches all parsed NPC behavior factories, the global **AttitudeMap** and **ItemAttitudeMap** which define faction relationships, and various configuration flags loaded from server settings. It also manages state for performance benchmarking tools.

-   **Thread Safety:** This class is **not** inherently thread-safe for all operations. While specific features like benchmarking use explicit locks (**ReentrantLock**) and atomic variables (**AtomicInteger**), the majority of its methods, especially those that interact with the ECS or the **BuilderManager**, are designed to be called from the main server game loop thread.

    **WARNING:** Accessing and modifying its internal state from asynchronous tasks or other threads can lead to race conditions, asset corruption, and server instability. All interactions with the ECS via this plugin must be scheduled on the appropriate world's execution context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | NPCPlugin | O(1) | Provides global singleton access to the plugin instance. |
| spawnNPC(...) | Pair | O(N) | Primary entry point for spawning an NPC of a given type at a specific location. Complexity depends on the role's initialization logic. Returns null if the role type is invalid. |
| getBuilderManager() | BuilderManager | O(1) | Returns the central manager for all NPC behavior builders. Use this to inspect available roles and behaviors. |
| getIndex(name) | int | O(1) | Translates a human-readable role name (e.g., "hytale:skeleton") into its internal integer index. |
| getName(index) | String | O(1) | Translates an internal role index back into its human-readable name. |
| getAttitudeMap() | AttitudeMap | O(1) | Retrieves the global map defining attitudes between different NPC groups. |
| startRoleBenchmark(...) | boolean | O(1) | Initiates a performance benchmark for NPC role ticking. |

## Integration Patterns

### Standard Usage

The most common interaction is to retrieve the singleton instance and use it to spawn an NPC in a specific world.

```java
// How a developer should normally use this
NPCPlugin plugin = NPCPlugin.get();
Store<EntityStore> worldEntityStore = world.getEntityStore().getStore();
Vector3d spawnPosition = new Vector3d(100, 64, 100);
Vector3f rotation = new Vector3f(0, 0, 0);

// Spawns an NPC defined by the "hytale:zombie" role asset
plugin.spawnNPC(worldEntityStore, "hytale:zombie", null, spawnPosition, rotation);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use **new NPCPlugin()**. The plugin is managed by the server's lifecycle. Always use the static **NPCPlugin.get()** method to obtain the instance.
-   **Asynchronous Spawning:** Do not call **spawnNPC** from an arbitrary thread. Spawning an entity involves modifying the world's ECS state, which must be done from the world's main thread to prevent concurrency issues.
-   **State Modification During Runtime:** Avoid directly modifying the internal state of the **BuilderManager** or other cached resources. These are managed by the plugin's asset lifecycle listeners. To change NPC behavior, modify the underlying asset files and allow the hot-reload mechanism to work.

## Data Pipeline

The NPCPlugin sits at the center of two primary data flows: asset loading and NPC behavior execution.

**Asset Loading Pipeline:**
> Flow:
> JSON/HOCON Asset File -> **AssetModule** (File Watcher) -> **LoadAssetEvent** -> **NPCPlugin** (Event Listener) -> **BuilderManager** -> In-Memory **Builder** Objects

**NPC Behavior Execution Pipeline (Simplified):**
> Flow:
> Game Tick -> **RoleSystems.BehaviourTickSystem** -> **NPCEntity**'s current **Role** -> **StateEvaluator** -> Selects **Instruction** -> **Action** / **BodyMotion** / **HeadMotion** -> Component changes (e.g., Velocity) -> Other ECS Systems (e.g., **ComputeVelocitySystem**) -> NPC acts in the world

