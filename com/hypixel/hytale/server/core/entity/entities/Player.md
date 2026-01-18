---
description: Architectural reference for the Player component, the server-side representation of a connected user.
---

# Player

**Package:** com.hypixel.hytale.server.core.entity.entities
**Type:** Stateful Component

## Definition
```java
// Signature
public class Player extends LivingEntity implements CommandSender, PermissionHolder, MetricProvider {
```

## Architecture & Concepts

The Player class is the primary server-side component representing a connected human player. It serves as the central authority for a player's gameplay state, distinguishing them from other non-player characters or creatures which are typically represented only by the base LivingEntity class.

This component is a critical part of the server's Entity Component System (ECS). It is not a standalone object but rather a data component attached to a generic Entity. It aggregates player-specific data and logic that builds upon the foundational features inherited from LivingEntity, such as health, attributes, and a basic inventory.

Key architectural roles include:

*   **State Aggregator:** It acts as an aggregate root for a player's complex state, holding direct ownership or references to specialized managers like WindowManager, HudManager, HotbarManager, and PageManager. This centralizes the management of all player-facing UI and interaction systems.
*   **System Integration Point:** By implementing interfaces like CommandSender and PermissionHolder, the Player component integrates the entity directly into the server's command and security systems. This allows the player entity to execute commands and have its permissions checked seamlessly.
*   **Network Bridge:** While not directly handling network I/O, it is intrinsically linked to a PlayerRef component. The Player class defines *what* state needs to be synchronized (e.g., GameMode, inventory changes), while the associated PlayerRef and its PacketHandler are responsible for the *how* of sending that state to the client.
*   **Persistence Contract:** The class defines a static CODEC field. This is a serialization contract used by the server's persistence layer (the Universe) to save and load player data between sessions. This includes everything from inventory and location to game mode and other configuration stored in PlayerConfigData.

## Lifecycle & Ownership

The Player component's lifecycle is strictly managed by the server's ECS and world systems. Direct manipulation of its lifecycle is a critical anti-pattern.

-   **Creation:** A Player component is instantiated under two main conditions:
    1.  When a new player joins the server for the first time, a new entity is created and a default Player component is attached.
    2.  When a returning player joins, their data is deserialized from storage via the Player.CODEC, re-hydrating the Player component with its last known state.
    This process is orchestrated by the server's core connection and world-loading logic, not by user code.

-   **Scope:** An instance of the Player component exists for the entire duration that a player is active within a World. Its data, managed within the PlayerConfigData object, persists across server restarts and player sessions. The in-memory component is ephemeral; the serialized data is persistent.

-   **Destruction:** The `remove` method is the designated cleanup entry point. It is invoked when a player disconnects or is administratively removed from the world. This method is responsible for a critical sequence of shutdown operations:
    1.  Clearing the player's loaded chunks via the ChunkTracker.
    2.  Removing the entity reference (PlayerRef) from the world's entity store.
    3.  Terminating the underlying network connection to the client.
    4.  Cancelling any pending asynchronous tasks, such as the client readiness timeout.

## Internal State & Concurrency

-   **State:** The Player component is highly mutable, representing the dynamic state of a player in the game world. It caches client-specific settings like `clientViewRadius`, tracks gameplay state like `gameMode`, and manages complex object graphs through its various sub-managers (WindowManager, etc.). State changes are frequent and are expected to be propagated to the client and/or persisted to storage.

-   **Thread Safety:** This component is **not thread-safe** and is designed to be accessed exclusively from its owning World's main thread. The presence of `world.isInThread()` checks followed by scheduling work via `world.execute()` is a clear indicator of this design. Modifying Player state from an external thread without proper scheduling will lead to race conditions, data corruption, and server instability.

    Certain fields, such as `waitingForClientReady` (AtomicReference) and `readyId` (AtomicInteger), use atomic wrappers. This is not for general-purpose thread safety, but to manage specific, isolated asynchronous operations related to the player connection lifecycle, ensuring that these specific state transitions occur safely. All business logic operating on the component must be synchronized with the main game loop.

## API Surface

The public API is designed for interaction from other server systems within the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| remove() | boolean | O(N) | Initiates the full cleanup and removal of the player from the world. Complexity depends on tracked chunks. |
| setGameMode(ref, gameMode, accessor) | static void | O(1) | Changes the player's game mode, triggering events, updating permissions, and sending network packets. |
| sendMessage(message) | void | O(1) | Sends a formatted chat or system message to the player's client. |
| handleClientReady(forced) | void | O(1) | Finalizes the player joining sequence. Fires the PlayerReadyEvent. |
| saveConfig(world, holder) | CompletableFuture | O(1) | Triggers the asynchronous saving of the player's configuration data to persistent storage. |
| applyMovementStates(ref, ...) | void | O(1) | Updates server-side movement capabilities (e.g., flying) based on client state and sends a confirmation packet. |
| hasPermission(id) | boolean | O(1) | Checks if the player has a specific permission string. Delegates to the PermissionsModule. |

## Integration Patterns

### Standard Usage

Direct interaction with a Player object instance is rare. The standard pattern is to operate on an entity's `Ref<EntityStore>` and use a `ComponentAccessor` to retrieve and modify the Player component within the context of an ECS system or event handler.

```java
// Correctly accessing the Player component within a system
Ref<EntityStore> playerEntityRef = ...; // Obtained from an event or query

// The accessor provides the correct context (world, etc.)
ComponentAccessor<EntityStore> accessor = ...; 

Player player = accessor.getComponent(playerEntityRef, Player.getComponentType());
if (player != null) {
    player.sendMessage(Message.literal("Hello from a server system!"));
}

// Correctly changing state via the static helper
Player.setGameMode(playerEntityRef, GameMode.Creative, accessor);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create a Player instance with `new Player()`. The server's entity management and persistence systems are solely responsible for its creation and lifecycle. Doing so will result in a disconnected, non-functional object.

-   **Cross-Thread Modification:** Do not get a Player component and modify its state from a separate thread (e.g., a database callback or a separate thread pool). This will bypass all safety checks. Schedule the modification on the appropriate world thread.
    ```java
    // BAD: Modifying state from another thread
    CompletableFuture.runAsync(() -> {
        Player p = accessor.getComponent(playerRef, Player.getComponentType());
        p.setFirstSpawn(false); // RACE CONDITION!
    });

    // GOOD: Scheduling the work on the world's thread
    world.execute(() -> {
        // This code now runs safely on the game loop tick
        Player p = accessor.getComponent(playerRef, Player.getComponentType());
        p.setFirstSpawn(false);
    });
    ```

-   **State Management without Packets:** Modifying a field like `gameMode` directly will desynchronize the server and client. Use designated methods like `setGameMode` which handle updating internal state, sending network packets, and firing events.

## Data Pipeline

The Player component is a nexus for data flowing between client input, server logic, and persistent storage.

**Example Flow: Player Spawning**

> Flow:
> Client Connection -> Server authenticates -> Universe.getPlayerStorage().load() -> **Player.CODEC deserializes data** -> Entity created with Player component -> Player joins World -> **Player.initGameMode()** -> **Player.startClientReadyTimeout()** -> Client sends ClientReadyPacket -> **Player.handleClientReady()** -> PlayerReadyEvent fired -> Player is fully active in the world.

**Example Flow: Game Mode Change**

> Flow:
> Server Command (`/gamemode creative`) -> Command System resolves target -> **Player.setGameMode()** is called -> ChangeGameModeEvent is fired -> Internal `gameMode` field is updated -> Permissions are updated -> `SetGameMode` packet is sent to client -> Client UI and physics rules update.

