---
description: Architectural reference for CreativeHubPlugin
---

# CreativeHubPlugin

**Package:** com.hypixel.hytale.builtin.creativehub
**Type:** Singleton

## Definition
```java
// Signature
public class CreativeHubPlugin extends JavaPlugin {
```

## Architecture & Concepts

The CreativeHubPlugin is a server-side extension that provides a framework for managing dynamic, temporary "hub" worlds. It acts as a high-level management layer on top of the core `InstancesPlugin`, specializing its generic world-spawning capabilities for the specific use case of player hubs.

Its primary architectural function is to intercept the player connection and world-transition lifecycle. By listening to critical server events like `PlayerConnectEvent`, the plugin can dynamically redirect players from a persistent "parent" world to a dedicated, on-demand hub instance. This allows for features like creative lobbies, mini-game waiting areas, or isolated build zones that are spawned only when needed and are associated with a specific entry point.

The plugin maintains a mapping between parent worlds and their active hub instances, ensuring that players from the same parent world are sent to the same shared hub. It also manages the data persistence required to return a player to their original location upon leaving the hub, using the `InstanceEntityConfig` and its own `CreativeHubEntityConfig` components.

A secondary but powerful feature is the ability to provision new *permanent* worlds from an instance template via the `spawnPermanentWorldFromTemplate` method. This elevates the plugin from a simple instance manager to a world provisioning tool, capable of stamping out new, persistent game worlds from a predefined asset.

## Lifecycle & Ownership

-   **Creation:** The CreativeHubPlugin is instantiated once by the server's plugin loader during the initial bootstrap sequence. The static singleton reference is set within the constructor, making it globally accessible via the static `get` method.
-   **Scope:** The plugin is a global singleton with a lifecycle tied directly to the server process. It persists for the entire server session. Its internal state, particularly the `activeHubInstances` map, holds references to `World` objects, which can prevent them from being unloaded as long as they are considered active hubs.
-   **Destruction:** The plugin is not explicitly destroyed. It is terminated and its resources are released only when the Hytale server process shuts down. Event listeners registered in the `setup` method are automatically deregistered by the core plugin system upon shutdown.

## Internal State & Concurrency

-   **State:** The plugin maintains significant, mutable state. Its core state is the `activeHubInstances` map, which acts as a cache of spawned hub worlds, keyed by the UUID of their parent world. This state is critical for routing players correctly and avoiding redundant world spawning.

-   **Thread Safety:** The class is designed to be thread-safe.
    -   The central `activeHubInstances` state is stored in a `ConcurrentHashMap`, which guarantees safe concurrent access and atomic operations.
    -   The `getOrSpawnHubInstance` method leverages the atomic `compute` method of `ConcurrentHashMap`. This prevents a critical race condition where multiple players from the same parent world could trigger the creation of multiple hub instances simultaneously.
    -   Event handlers are static methods that operate on the singleton instance. As they are invoked by the server's multithreaded event bus, their reliance on the `ConcurrentHashMap` is essential for maintaining data consistency.

    **WARNING:** The `getOrSpawnHubInstance` method contains a blocking `.join()` call on a `CompletableFuture`. Invoking this method from a time-sensitive thread, such as the main server tick loop, will cause severe performance degradation and server stalls while the hub world is being created.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | CreativeHubPlugin | O(1) | Retrieves the global singleton instance. |
| getOrSpawnHubInstance(World, CreativeHubWorldConfig, Transform) | World | O(N) | Atomically retrieves an existing hub or spawns a new one. **This is a blocking operation** that can take seconds to complete. |
| getActiveHubInstance(World) | World | O(1) | Retrieves the active hub for a parent world, if one exists and is alive. Returns null otherwise. |
| clearHubInstance(UUID) | void | O(1) | Removes the mapping for a parent world, allowing its hub instance to be garbage collected. |
| spawnPermanentWorldFromTemplate(String, String) | CompletableFuture<World> | O(N) | Asynchronously creates a new, persistent world from an instance template. Involves heavy file I/O. |

## Integration Patterns

### Standard Usage

The plugin is primarily driven by server events. For direct interaction, another plugin would retrieve the singleton and use it to manage hub instances, often in response to a custom command or interaction.

```java
// Example: Manually forcing a player into a hub world
CreativeHubPlugin hubPlugin = CreativeHubPlugin.get();
PlayerRef player = ...;
World currentWorld = player.getWorld();

// Assumes the current world is configured with CreativeHubWorldConfig
CreativeHubWorldConfig hubConfig = CreativeHubWorldConfig.get(currentWorld.getWorldConfig());
if (hubConfig != null) {
    // WARNING: This is a blocking call and should not be run on the main thread.
    Transform returnPoint = player.getTransform();
    World hubInstance = hubPlugin.getOrSpawnHubInstance(currentWorld, hubConfig, returnPoint);
    player.teleport(hubInstance.getWorldConfig().getSpawnProvider().getSpawnPoint(hubInstance, player.getUuid()));
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new CreativeHubPlugin()`. The server's plugin loader is solely responsible for its creation. Always use the static `CreativeHubPlugin.get()` accessor.
-   **Blocking the Main Thread:** Do not call `getOrSpawnHubInstance` from any synchronous, performance-critical code path like an entity update tick. The internal `.join()` will freeze the calling thread until the entire world is spawned and loaded.
-   **State Mismanagement:** Do not manually add or remove entries from `activeHubInstances`. Use the provided API methods like `clearHubInstance` to ensure proper lifecycle management. The plugin's event listeners handle most state changes automatically.

## Data Pipeline

The plugin's most common data flow is the interception of a player joining the server.

> Flow:
> Player Connection Attempt -> Server fires `PlayerConnectEvent` -> Event Bus invokes `CreativeHubPlugin.onPlayerConnect` -> Plugin inspects target world for `CreativeHubWorldConfig` -> **`getOrSpawnHubInstance`** is called -> `InstancesPlugin` is invoked to copy and load the world template -> A new `World` object is returned -> The `PlayerConnectEvent` is mutated with `event.setWorld(hubInstance)` -> The core server logic places the player in the newly spawned hub world instead of their original destination.

