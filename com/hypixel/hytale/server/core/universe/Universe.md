---
description: Architectural reference for Universe
---

# Universe

**Package:** com.hypixel.hytale.server.core.universe
**Type:** Singleton

## Definition
```java
// Signature
public class Universe extends JavaPlugin implements IMessageReceiver, MetricProvider {
```

## Architecture & Concepts

The Universe is the highest-level container for the server's active game state. It functions as a session-scoped singleton that owns and manages all active **World** instances and connected **PlayerRef** entities. Architecturally, it serves as the critical bridge between the server's core infrastructure (networking, configuration, plugins) and the dynamic game simulation.

Its primary responsibilities include:

*   **World Lifecycle Management:** Orchestrates the loading, initialization, runtime management, and shutdown of all game worlds. It reads world configurations from disk and constructs the corresponding `World` objects, each running in its own thread.
*   **Player Session Management:** Manages the entire lifecycle of a player's connection. It handles the `addPlayer` flow upon connection, which involves loading player data, creating the player entity, and injecting it into a target world. It also processes player disconnection via `removePlayer`.
*   **Central Registry:** Acts as the authoritative source for locating active worlds (by name or UUID) and players (by UUID or username). All game systems that need to interact with a player or world not in their immediate context must query the Universe.
*   **Global Event Hub:** Listens for server-wide events, such as `ShutdownEvent`, to trigger global actions like disconnecting all players and shutting down all worlds in an orderly sequence.
*   **Global Communication Bus:** Implements `IMessageReceiver` to provide methods for broadcasting packets and messages to all connected players, such as with `broadcastPacket`.

The Universe guarantees that core collections like the list of worlds and players are safe for concurrent access, but delegates the thread safety of the simulation itself to the individual `World` instances.

### Lifecycle & Ownership

-   **Creation:** The Universe is instantiated once by the `PluginManager` during the server's initial bootstrap sequence. Its singleton instance is set in the constructor and is accessible globally via the static `Universe.get()` method.
-   **Scope:** The Universe object persists for the entire lifetime of the server process. It is one of the first core plugins to start and one of the last to shut down.
-   **Destruction:** The `shutdown()` method is invoked by the `PluginManager` during the server shutdown sequence. This method is responsible for forcefully disconnecting all players and stopping all running worlds to ensure a clean exit and prevent data corruption.

A critical lifecycle gate is the `universeReady` `CompletableFuture`. This future only completes after all worlds designated for startup have been loaded and initialized. Systems that depend on a fully available set of worlds **must** wait for this future to complete before proceeding.

## Internal State & Concurrency

-   **State:** The Universe maintains a highly mutable state, primarily consisting of concurrent maps for `worlds` and `players`. These collections are constantly modified as worlds are loaded/unloaded and players connect/disconnect.
-   **Thread Safety:** This class is designed to be thread-safe for its primary responsibilities. The use of `ConcurrentHashMap` for `worlds` and `players` collections allows for safe addition and removal of elements from multiple threads (e.g., a network thread adding a player, a command thread adding a world).

    **WARNING:** While the collections themselves are thread-safe, the objects they contain (`World`, `PlayerRef`) are not inherently safe to be modified from any thread. All interactions that modify the internal state of a `World` or an entity within it **must** be executed on that world's dedicated thread. The Universe manages the collections, not the internal simulation logic of its children.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | Universe | O(1) | Returns the global singleton instance. |
| addWorld(name) | CompletableFuture<World> | O(N) | Asynchronously creates and loads a new world. Involves disk I/O and is computationally expensive. |
| loadWorld(name) | CompletableFuture<World> | O(N) | Asynchronously loads an existing world from disk. |
| removeWorld(name) | boolean | O(N) | Triggers the shutdown and removal of a world. Dispatches a `RemoveWorldEvent` which can be cancelled. |
| getWorld(name / uuid) | World | O(1) | Retrieves a loaded world by its name or UUID. Returns null if not found. |
| addPlayer(...) | CompletableFuture<PlayerRef> | O(N) | Handles the full player connection pipeline, from data loading to injection into a world. Highly complex. |
| removePlayer(playerRef) | void | O(N) | Initiates the player removal process, which is completed asynchronously on the player's current world thread. |
| getPlayer(uuid / name) | PlayerRef | O(1) / O(N) | Retrieves a connected player by UUID (fast) or by name (slower). |
| broadcastPacket(packet) | void | O(N) | Sends a network packet to every connected player. |

## Integration Patterns

### Standard Usage

The Universe is almost always accessed via its static singleton accessor. It is the primary entry point for obtaining references to worlds and players.

```java
// How a developer should normally use this
Universe universe = Universe.get();

// Find the default world
World defaultWorld = universe.getDefaultWorld();
if (defaultWorld == null) {
    // Handle case where no world is available
    return;
}

// Find a specific player
PlayerRef player = universe.getPlayer(someUuid);
if (player != null) {
    player.sendMessage(Message.text("Hello from the Universe!"));
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The Universe is a plugin managed by the server's lifecycle. Never attempt to create an instance with `new Universe()`. Always use `Universe.get()`.
-   **Ignoring Futures:** Methods like `addWorld` and `addPlayer` are asynchronous and return a `CompletableFuture`. Code that depends on the result of these operations must block or chain callbacks on the future. Assuming the operation is complete immediately will lead to race conditions and `NullPointerException`.
-   **Modifying World Collections:** The map returned by `getWorlds()` is explicitly unmodifiable. Attempting to add or remove worlds from this collection will throw an `UnsupportedOperationException`. Use the `addWorld` and `removeWorld` methods instead.
-   **Cross-Thread World Modification:** Do not get a `World` object from the Universe and immediately begin modifying its internal state from the calling thread. All simulation logic must be dispatched to the world's own executor.

## Data Pipeline

The Universe is central to the data flow for a player joining the server. It coordinates with storage, networking, and the entity-component system to bring a player into the game.

> Flow: Player Join
>
> Netty Channel Active -> Login Packet -> **Universe.addPlayer()** -> PlayerStorage.load(uuid) -> Holder<EntityStore> created -> PlayerRef & Components attached -> PlayerConnectEvent dispatched -> Target World selected -> **World.addPlayer()** -> Player entity added to world simulation -> PlayerRef added to Universe.players map -> Client receives join confirmation packets.

