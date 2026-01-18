---
description: Architectural reference for InstancesPlugin
---

# InstancesPlugin

**Package:** com.hypixel.hytale.builtin.instances
**Type:** Singleton

## Definition
```java
// Signature
public class InstancesPlugin extends JavaPlugin {
```

## Architecture & Concepts
The InstancesPlugin is the central authority for managing temporary, isolated game worlds known as "instances". It provides a complete lifecycle management system for creating worlds from predefined templates, transitioning players into and out of them, and safely tearing them down when they are no longer needed.

This system acts as a foundational layer for features like dungeons, minigames, or player housing. It is not merely a world loader; it is a comprehensive orchestration engine that integrates deeply with several core server systems:

*   **Asset System:** Instances are defined as assets on the server's filesystem. The plugin is responsible for locating, reading, and preparing these assets for runtime use.
*   **Universe & World Management:** It leverages the `Universe` service to dynamically spawn new `World` objects from the instance assets. Each instance runs as a fully-fledged world, typically on its own thread.
*   **Component System:** The plugin registers a suite of components and resources to manage instance-related state.
    *   **InstanceWorldConfig:** A plugin-specific configuration attached to a `WorldConfig` that stores metadata like return points and automatic removal conditions.
    *   **InstanceEntityConfig:** A component attached to players that tracks their return point when they enter an instance.
    *   **InstanceDataResource:** A world-scoped resource used by the `RemovalSystem` to track instance state, such as whether a player has ever entered it.
*   **Event Bus:** It subscribes to global player events (`PlayerConnectEvent`, `AddPlayerToWorldEvent`) to transparently manage player state and handle complex reconnection scenarios involving instances.

The core architectural pattern is the **dynamic instantiation of world templates**. An "instance asset" is a blueprint on disk. The `spawnInstance` method transforms this blueprint into a live, interactive `World` object, handling file copying, configuration patching, and registration with the `Universe`.

### Lifecycle & Ownership
-   **Creation:** A single instance of InstancesPlugin is created by the server's plugin loader during the bootstrap sequence. The static `instance` field is set within the constructor, enforcing the singleton pattern.
-   **Scope:** The plugin object exists for the entire lifetime of the server process. It is a persistent, core service.
-   **Destruction:** The plugin is unloaded and eligible for garbage collection only during a full server shutdown.

## Internal State & Concurrency
-   **State:** The InstancesPlugin class itself is largely stateless. Its fields, such as `instanceDataResourceType`, are immutable references to type definitions that are assigned once during the `setup` phase. All mutable state related to active instances or players within them is stored externally in components attached to `World` and `EntityStore` objects.
-   **Thread Safety:** This class is designed to be thread-safe. Public methods that perform heavy lifting, like `spawnInstance`, are asynchronous and return a `CompletableFuture`. They delegate work to the `Universe` and `World` systems, which manage their own threading models. Event handlers are invoked by the event bus on the appropriate world thread, ensuring safe access to world-specific data. Direct calls to its methods from any thread are safe.

## API Surface
The public API provides high-level orchestration for the entire instance lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spawnInstance(name, forWorld, returnPoint) | CompletableFuture<World> | O(N) | Asynchronously creates a new instance from an asset. N is the size of the asset files. This is the primary entry point for creating instances. |
| teleportPlayerToLoadingInstance(entityRef, accessor, worldFuture, overrideReturn) | void | O(1) | Safely teleports a player to an instance that is still being created. Manages state transitions and failure recovery. |
| teleportPlayerToInstance(playerRef, accessor, targetWorld, overrideReturn) | void | O(1) | Teleports a player to an already-existing instance by attaching a `Teleport` component. |
| exitInstance(targetRef, accessor) | void | O(1) | Ejects a player from an instance, returning them to their designated return point. |
| safeRemoveInstance(world) | void | O(1) | Marks an active instance for automatic cleanup. The `RemovalSystem` will destroy the world based on configured conditions (e.g., when all players leave). |
| getInstanceAssets() | List<String> | O(F) | Scans the asset packs and returns a list of all discoverable instance template names. F is the number of files in asset directories. |
| loadInstanceAssetForEdit(name) | CompletableFuture<World> | O(N) | Spawns a special, non-interactive version of an instance for in-game editing. |

## Integration Patterns

### Standard Usage
The standard flow involves spawning an instance and then teleporting a player to the future result. This non-blocking pattern is critical for server performance.

```java
// Standard flow: Spawn an instance and teleport a player asynchronously.
InstancesPlugin plugin = InstancesPlugin.get();
World currentWorld = player.getWorld();
Transform returnPoint = player.getTransform();

// 1. Begin spawning the instance. This is non-blocking.
CompletableFuture<World> instanceFuture = plugin.spawnInstance("dungeon-level-1", currentWorld, returnPoint);

// 2. Immediately schedule the player's teleport to the future world.
//    The system handles the transition once the world is ready.
InstancesPlugin.teleportPlayerToLoadingInstance(playerRef, accessor, instanceFuture, null);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new InstancesPlugin()`. The server's plugin loader is solely responsible for its creation. Always use the static `InstancesPlugin.get()` accessor to retrieve the singleton instance.
-   **Blocking on Futures:** Calling `join()` or `get()` on the `CompletableFuture` returned by `spawnInstance` from a world's main thread will freeze that world until the instance is fully loaded, causing severe lag. Always use asynchronous chaining (e.g., `thenCompose`, `whenComplete`).
-   **Manual World Removal:** Do not call `Universe.get().removeWorld()` on an instance world directly. This bypasses critical cleanup logic and can lead to corrupted player data or memory leaks. Always use `InstancesPlugin.safeRemoveInstance()` to allow the `RemovalSystem` to perform a graceful shutdown.

## Data Pipeline
The process of creating an instance and moving a player into it follows a well-defined, asynchronous pipeline.

> **Flow: Instance Creation & Player Teleport**
>
> 1.  **API Call:** An external system calls `spawnInstance("my_dungeon", ...)`.
> 2.  **Asset Resolution:** The plugin searches asset packs for `Server/Instances/my_dungeon`.
> 3.  **Filesystem I/O:** The instance asset files, including `instance.bson` and world data, are copied to a new, unique world directory (e.g., `worlds/instance-my_dungeon-uuid`).
> 4.  **Configuration:** The `WorldConfig` is loaded from `instance.bson` and patched with runtime data (a new UUID, the return point, etc.).
> 5.  **World Creation:** The `Universe` is instructed to create and load a new `World` from the prepared directory and configuration. This happens on a worker thread.
> 6.  **Player Transition:** Simultaneously, `teleportPlayerToLoadingInstance` is called.
> 7.  **State Management:** The player's `InstanceEntityConfig` component is updated with the return point. The player is then removed from their current world.
> 8.  **Synchronization:** The system waits for the `CompletableFuture<World>` from step 5 to complete.
> 9.  **Player Injection:** Once the new instance `World` is ready, `world.addPlayer()` is called, injecting the player into the new world and sending the necessary packets to the client.
> 10. **Failure Recovery:** If any step in the world creation future fails, the logic in `whenComplete` catches the exception and attempts to re-add the player to their original world to prevent them from being stuck in limbo.

