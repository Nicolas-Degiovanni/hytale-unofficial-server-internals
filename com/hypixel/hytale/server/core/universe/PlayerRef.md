---
description: Architectural reference for PlayerRef
---

# PlayerRef

**Package:** com.hypixel.hytale.server.core.universe
**Type:** Component / Session-Scoped

## Definition
```java
// Signature
public class PlayerRef implements Component<EntityStore>, MetricProvider, IMessageReceiver {
```

## Architecture & Concepts

The PlayerRef class is the central server-side representation of a connected player. It is not the player *entity* itself, but rather a high-level handle or reference that bridges the network layer with the game simulation. It functions as a stateful component within the server's Entity Component System (ECS), implementing the Component interface.

Its primary architectural role is to manage the lifecycle of a player's presence within the game universe. It holds direct references to critical player-specific services, including the PacketHandler for network communication and the ChunkTracker for world data streaming.

A key concept embodied by this class is the separation between a connected player session and the player's in-world entity. A PlayerRef exists for the entire duration of a player's connection, but its associated entity may be added to or removed from a World's EntityStore. This is managed via an internal state machine that transitions between a live entity reference (Ref) and a serialized entity representation (Holder). This abstraction is critical for features like world transfers and instancing, allowing the player's session to persist while their physical manifestation is moved between simulation environments.

## Lifecycle & Ownership

-   **Creation:** A PlayerRef instance is created by the server's connection management system upon successful authentication of a player. The constructor is injected with session-specific dependencies like the PacketHandler, establishing the link to a unique network connection.
-   **Scope:** The object's lifetime is strictly tied to the player's network session. It persists as long as the player is connected to the server, regardless of whether they are loaded into a world or sitting in a lobby.
-   **Destruction:** The PlayerRef is marked for garbage collection when the player's session terminates (e.g., disconnection). The server's session manager is responsible for clearing all strong references to the PlayerRef instance, allowing it to be reclaimed. The isValid method can be used to check if the object is still tied to a valid session.

## Internal State & Concurrency

-   **State:** PlayerRef is a highly mutable, stateful object. It caches the player's UUID, username, and language. Critically, its internal state tracks whether the player is currently active in a world via the nullable *entity* (a live Ref) and *holder* (a serialized representation) fields. It also caches the last known position and rotation.
-   **Thread Safety:** This class is **not thread-safe** and mandates strict thread discipline. Operations that modify the player's state within a world, such as addToStore and removeFromStore, contain `assertThread()` checks. These methods must be called exclusively from the main thread of the target World. Accessing ECS components via getComponent from an external thread is explicitly disallowed and will log a severe error, as it can lead to race conditions and data corruption. The implementation attempts to recover from such violations by dispatching the call to the correct thread via a CompletableFuture, but this is a fallback and not a substitute for correct thread management.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addToStore(Store) | Ref | O(N) | Adds the player's entity to a world's EntityStore. Throws IllegalStateException if already in a world. Must be called on the world's main thread. |
| removeFromStore() | Holder | O(N) | Removes the player's entity from its current world, returning a serialized Holder. Throws IllegalStateException if not in a world. Must be called on the world's main thread. |
| isValid() | boolean | O(1) | Checks if the player reference is still valid (i.e., the session is active). |
| getReference() | Ref | O(1) | Returns the live entity reference if the player is currently in a world, otherwise null. |
| getPacketHandler() | PacketHandler | O(1) | Provides direct access to the player's network packet handler for sending data. |
| updatePosition(World, Transform, Vector3f) | void | O(1) | Updates the cached transform and head rotation. Expected to be called frequently from the server tick. |
| referToServer(String, int, byte[]) | void | O(1) | Instructs the client to disconnect and connect to a different server. Triggers an outbound ClientReferral packet. |
| sendMessage(Message) | void | O(1) | Sends a formatted chat message to the player. |

## Integration Patterns

### Standard Usage

Game logic systems should retrieve a PlayerRef from a central player registry or service. It is the primary entry point for interacting with a specific player, from sending them a message to initiating a world transfer.

```java
// Example: Moving a player to a new world
PlayerRef player = playerRegistry.findByUuid(someUuid);
World currentWorld = universe.getWorld(player.getWorldUuid());
World newWorld = universe.getWorld(newWorldUuid);

// Must be executed on the current world's thread
Holder<EntityStore> playerHolder = currentWorld.execute(() -> {
    return player.removeFromStore();
});

// Must be executed on the new world's thread
newWorld.execute(() -> {
    player.replaceHolder(playerHolder); // Update the holder before adding
    player.addToStore(newWorld.getStore());
});
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create a PlayerRef using `new PlayerRef()`. It is the responsibility of the core connection and session management systems to construct and initialize these objects with the correct dependencies.
-   **Cross-Thread State Modification:** Do not call `addToStore` or `removeFromStore` from any thread other than the target world's main simulation thread. This will fail runtime assertions and destabilize the server.
-   **Asynchronous Component Access:** Avoid calling `getComponent` from an asynchronous task or a different world's thread. While the system has a slow fallback to prevent a crash, this indicates a severe architectural flaw and will be logged as an error. All entity interactions should be dispatched to the correct thread.

## Data Pipeline

The PlayerRef acts as a bidirectional hub, routing data between the network and the game simulation.

> **Outbound Flow (e.g., Chat Message):**
> Game System -> `PlayerRef.sendMessage(message)` -> `PlayerRef.getPacketHandler().writeNoCache(packet)` -> Network IO Thread -> Client

> **State Synchronization Flow:**
> World Tick Thread -> `PlayerRef.updatePosition(...)` -> Internal State (transform) Updated -> (Separately) Position Update Packet sent via PacketHandler -> Client

