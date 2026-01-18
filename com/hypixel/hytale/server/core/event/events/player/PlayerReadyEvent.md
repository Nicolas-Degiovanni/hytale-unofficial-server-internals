---
description: Architectural reference for PlayerReadyEvent
---

# PlayerReadyEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Value Object

## Definition
```java
// Signature
public class PlayerReadyEvent extends PlayerEvent<String> {
```

## Architecture & Concepts
The PlayerReadyEvent is an immutable data carrier that signals a critical state transition in the player connection lifecycle. It serves as a message within the server's event-driven architecture, indicating that a client has completed all necessary loading and handshaking and is now prepared to be fully integrated into the game world.

This class decouples the low-level network and connection management systems from the high-level game logic systems. The component that receives the final "ready" packet from a client is responsible only for creating and publishing this event. It does not need to know which systems must react. Conversely, systems like the WorldManager or PlayerSpawner can subscribe to this event to perform their duties without being tightly coupled to the network layer.

The inclusion of a `readyId` suggests a request-response or synchronization mechanism. The server may issue a challenge or token that the client must return to be considered "ready", preventing state mismatches or replay attacks.

## Lifecycle & Ownership
- **Creation:** Instantiated by a session or connection manager deep within the server's networking layer. This occurs precisely when the server receives the final confirmation packet from a client signifying it is ready to enter the world.

- **Scope:** Transient and extremely short-lived. An instance of PlayerReadyEvent exists only for the duration of its dispatch through the server's central EventBus.

- **Destruction:** The object has no explicit destruction logic. Once all subscribed listeners have processed the event, it falls out of scope and is reclaimed by the Java Garbage Collector. Its lifecycle is entirely managed by the event dispatch system.

## Internal State & Concurrency
- **State:** Strictly immutable. All internal fields, including the inherited Player and EntityStore references, are declared final and are set exclusively during construction. This design guarantees that the event's data is consistent and cannot be altered during its dispatch.

- **Thread Safety:** Inherently thread-safe due to its immutability. This object can be safely passed across thread boundaries without any need for locks or other synchronization primitives. This is a critical feature for a high-performance, multi-threaded server environment where events may be processed by a pool of worker threads.

## API Surface
The public contract is minimal, focused on providing read-only access to the event's context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getReadyId() | int | O(1) | Returns the unique identifier for this ready state transition. |
| getPlayer() | Player | O(1) | *Inherited from PlayerEvent.* Retrieves the player entity associated with this event. |
| getEntityStoreRef() | Ref<EntityStore> | O(1) | *Inherited from PlayerEvent.* Provides a reference to the world context. |

## Integration Patterns

### Standard Usage
This event is not intended to be created by general game logic developers. Instead, systems subscribe to it via the server's EventBus to react when a player becomes ready.

```java
// Example of a listener in a world management service
@Subscribe
public void onPlayerReady(PlayerReadyEvent event) {
    Player player = event.getPlayer();
    int readyId = event.getReadyId();

    // Log the transition for auditing
    log.info("Player {} is now ready with readyId {}. Spawning into world.", player.getName(), readyId);

    // Trigger player spawning logic
    this.worldSpawner.spawnPlayerInWorld(player);

    // Notify other systems that the player is now active
    this.gameStateManager.setPlayerAsActive(player);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Dispatch:** Never instantiate and pass this event directly to another service. Always publish it to the global server EventBus. Bypassing the bus breaks the decoupling principle and can lead to systems missing a critical state change, leaving the player in a broken or "limbo" state.
- **State Assumption:** Do not assume the player entity is already spawned in the world when you receive this event. This event is often the *trigger* for the spawning process itself. Always check the player's state before acting.

## Data Pipeline
The PlayerReadyEvent is a key message in the player connection data flow, translating a network signal into a server-wide logical event.

> Flow:
> Client "Ready" Packet -> Server Network Layer -> SessionManager -> **new PlayerReadyEvent()** -> Server EventBus -> (WorldSpawner, QuestSystem, etc.) -> Player Entity Spawned in World

