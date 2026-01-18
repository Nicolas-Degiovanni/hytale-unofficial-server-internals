---
description: Architectural reference for ServerPlayerListModule
---

# ServerPlayerListModule

**Package:** com.hypixel.hytale.server.core.modules.serverplayerlist
**Type:** Singleton (Core Plugin)

## Definition
```java
// Signature
public class ServerPlayerListModule extends JavaPlugin {
```

## Architecture & Concepts
The ServerPlayerListModule is a foundational server-side system responsible for synchronizing the state of the in-game player list with all connected clients. It functions as a reactive, event-driven bridge between the server's central player state management (the Universe) and the client-side user interface.

Its core responsibilities are:
1.  **Full State Synchronization on Join:** When a new player connects, this module sends them a complete snapshot of all currently connected players.
2.  **Delta Updates:** It broadcasts incremental updates to all clients when a player connects, disconnects, or changes their world.
3.  **Periodic State Refresh:** It runs a scheduled task to periodically broadcast non-critical state, such as player ping, to all clients.

This module is designed to be stateless. It does not maintain its own copy of the player list. Instead, for every operation, it queries the Universe component, which serves as the single source of truth for all player data. This design prevents state desynchronization within the server itself and simplifies the module's logic.

## Lifecycle & Ownership
-   **Creation:** The ServerPlayerListModule is instantiated by the server's core plugin loader during the bootstrap sequence. Its `MANIFEST` declares a dependency on the Universe, ensuring it is initialized only after the primary player state repository is available.
-   **Scope:** This is a server-wide singleton that persists for the entire runtime of the server process. Its lifecycle is directly tied to the server's lifecycle.
-   **Destruction:** The module is destroyed and its resources are released during the standard server shutdown procedure, managed by the plugin system.

## Internal State & Concurrency
-   **State:** This class is intentionally stateless. It holds no mutable state regarding the list of players. All player information is fetched on-demand from the Universe service. This makes the module highly robust and resilient to state corruption.

-   **Thread Safety:** The module's design assumes a specific threading model.
    -   Event handlers such as onPlayerConnect and onPlayerDisconnect are invoked by the EventRegistry, which is expected to operate on the main server thread. Operations within these handlers are therefore thread-safe by convention.
    -   The `broadcastPingUpdates` method is executed on the `HytaleServer.SCHEDULED_EXECUTOR`, a separate thread pool. It performs a read-only iteration of the player list from the Universe. This is only safe if the underlying collection in Universe is thread-safe for concurrent reads, which is a critical assumption for system stability.

    **Warning:** Direct modification of the Universe player list from an arbitrary thread could lead to a `ConcurrentModificationException` or other race conditions within the `broadcastPingUpdates` task. All modifications to the global player list must be synchronized or performed on the main server thread.

## API Surface
The public API is minimal, as this module is not intended for direct interaction by other plugins. Its functionality is triggered exclusively by internal server events.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ServerPlayerListModule | O(1) | Retrieves the static singleton instance of the module. |

## Integration Patterns

### Standard Usage
This module is a core system service and is not designed to be used directly. It is loaded automatically by the server and functions autonomously by listening to events on the global EventRegistry. Developers do not need to, and should not, interact with this class.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ServerPlayerListModule()`. The server's plugin loader is solely responsible for its creation and lifecycle.
-   **Manual Invocation:** Do not retrieve the instance via `get()` to manually call its event handlers. This would bypass the event bus and could lead to unpredictable behavior and state inconsistencies.
-   **State Reliance:** Do not build features that depend on the precise timing of player list updates. Packet delivery is asynchronous and subject to network latency.

## Data Pipeline
The module translates server-side events into network packets sent to clients. The data flow is unidirectional from server to client.

**Flow for a New Player Connecting:**
> Player TCP Connection Established -> `PlayerConnectEvent` Fired -> EventRegistry -> **ServerPlayerListModule::onPlayerConnect** -> Reads full player list from `Universe` -> Constructs `AddToServerPlayerList` packets -> `PacketHandler.write()` -> Network Layer -> All Connected Clients

**Flow for a Periodic Ping Update:**
> `HytaleServer.SCHEDULED_EXECUTOR` Timer Fires -> **ServerPlayerListModule::broadcastPingUpdates** -> Reads full player list from `Universe` -> Calculates ping for each player -> Constructs `UpdateServerPlayerListPing` packet -> `PacketHandler.writeNoCache()` -> Network Layer -> All Connected Clients

