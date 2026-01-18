---
description: Architectural reference for MountPlugin
---

# MountPlugin

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Singleton

## Definition
```java
// Signature
public class MountPlugin extends JavaPlugin {
```

## Architecture & Concepts
The MountPlugin serves as the central initialization and configuration hub for the entire server-side mounting system. As a subclass of JavaPlugin, it is automatically discovered and loaded by the server's plugin manager during the bootstrap sequence. Its primary architectural role is not to manage runtime game state, but to register all the necessary building blocks of the mounting feature with the core engine.

This class acts as a manifest for the feature, performing the following critical registrations:

*   **Entity Components:** It defines and registers several core components with the Entity Component System (ECS) registry, such as NPCMountComponent, MountedComponent, and MinecartComponent. These components are the data containers that define mount-related state on entities.
*   **ECS Systems:** It registers a comprehensive suite of systems (e.g., MountSystems.TrackerUpdate, NPCMountSystems.DismountOnPlayerDeath) that contain the logic for processing entities that possess mount-related components. These systems are executed by the main game loop and are responsible for all behavioral aspects of mounting.
*   **Interaction Handlers:** It registers custom interaction types like MountInteraction and SeatingInteraction, allowing specific player actions (e.g., right-clicking an entity) to be decoded and processed as mount-related events.
*   **Network Handlers:** It registers the MountGamePacketHandler to process custom network packets sent from the client, enabling client-driven mount actions like input control.
*   **Event Listeners:** It subscribes to global server events, such as PlayerDisconnectEvent, to ensure state integrity and perform necessary cleanup when players leave the game.
*   **Commands:** It registers console and in-game commands, like MountCommand, for administrative or debugging purposes.

In essence, the MountPlugin is the single entry point that weaves the mounting feature into the fabric of the server engine.

### Lifecycle & Ownership
- **Creation:** The MountPlugin is instantiated exactly once by the server's plugin loader during the server startup process. The constructor receives a JavaPluginInit object, which provides the necessary context and access to server-wide registries.

- **Scope:** The object's lifecycle is tied directly to the server's runtime. It persists as a singleton, accessible via the static MountPlugin.getInstance method, for the entire duration the server is online.

- **Destruction:** The instance is destroyed and its resources are released when the server shuts down or if the plugin is explicitly unloaded by an administrator.

## Internal State & Concurrency
- **State:** The internal state of a MountPlugin instance consists almost entirely of immutable references to ComponentType objects. These references are established once within the setup method and do not change during the plugin's lifetime. The class itself is stateless regarding dynamic game world information; all such state is stored in ECS components within the world.

- **Thread Safety:** This class is designed to be thread-safe.
    - The setup method is guaranteed by the engine to be called from a single main thread during server initialization, preventing race conditions during registration.
    - The static utility methods, such as onPlayerDisconnect, are designed to handle events from potentially different threads. They achieve safety by scheduling state-mutating logic to be run on the appropriate World thread via world.execute. This pattern ensures that all modifications to ECS components occur in a serialized, thread-safe manner within the game loop's context.

    **Warning:** While the plugin itself is thread-safe, direct modification of components retrieved via its accessors must adhere to the engine's threading model, typically by executing logic on the corresponding world thread.

## API Surface
The public API is minimal, primarily exposing static utility functions and accessors for the component types it registers. Direct interaction with a MountPlugin instance is uncommon; developers typically interact with the components and systems it enables.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | static MountPlugin | O(1) | Retrieves the singleton instance of the plugin. |
| getMountComponentType() | ComponentType | O(1) | Returns the registered type for the NPCMountComponent. |
| getMountedComponentType() | ComponentType | O(1) | Returns the registered type for the MountedComponent. |
| checkDismountNpc(store, player) | static void | O(N) | Checks if a player is mounted and initiates the dismount sequence if necessary. Complexity depends on entity lookup. |
| dismountNpc(store, entityId) | static void | O(N) | Core logic to dismount a player from an NPC, resetting roles and movement settings. Complexity depends on entity lookup. |
| resetOriginalPlayerMovementSettings(playerRef, store) | static void | O(1) | Resets a player's movement controller to its default state and notifies the client. |

## Integration Patterns

### Standard Usage
Developers should not interact with the MountPlugin directly for game logic. Instead, they interact with the systems it registers by adding or removing its components from entities. The primary use of the class is to retrieve the singleton to access a registered ComponentType.

```java
// Correctly adding a mount component to an entity
// This triggers the systems registered by MountPlugin

MountPlugin plugin = MountPlugin.getInstance();
ComponentType<EntityStore, NPCMountComponent> type = plugin.getMountComponentType();

// Within a world-threaded context...
world.getSystem(ComponentSystem.class).addComponent(entityRef, type);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new MountPlugin(). The server's plugin loader is solely responsible for its creation. Attempting to do so will break the singleton pattern and lead to an uninitialized, non-functional object.

- **Calling Setup Manually:** The setup method is part of the plugin lifecycle and must only be invoked by the engine. Calling it manually will result in duplicate registrations and severe server instability.

- **Assuming Initialization:** Do not attempt to access component types from getInstance during the earliest phases of server startup. The instance is only fully configured after its setup method has completed.

## Data Pipeline
The following illustrates the data flow for a critical cleanup path: a player disconnecting while mounted on an NPC.

> Flow:
> Player TCP Connection Drops -> Server Network Layer fires **PlayerDisconnectEvent** -> MountPlugin.onPlayerDisconnect (static listener) -> Schedules task on World Thread -> **MountPlugin.checkDismountNpc** -> **MountPlugin.dismountNpc** -> RoleChangeSystem.requestRoleChange & store.removeComponent -> Player and NPC components are reset to their pre-mount state.

