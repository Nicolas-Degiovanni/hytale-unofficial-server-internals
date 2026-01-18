---
description: Architectural reference for PlayerSetupDisconnectEvent
---

# PlayerSetupDisconnectEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Event Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class PlayerSetupDisconnectEvent implements IEvent<Void> {
```

## Architecture & Concepts
The PlayerSetupDisconnectEvent is an immutable data carrier object that signals a critical failure state within the server's connection lifecycle. It is dispatched when a player's connection is terminated *during the initial setup and authentication phase*, before a complete player entity has been established in the game world.

This event is a cornerstone of the server's decoupled, event-driven architecture. It allows the low-level network layer, specifically the PacketHandler, to communicate connection failures to higher-level systems—such as authentication, session management, and logging services—without creating direct dependencies.

By implementing the IEvent interface with a Void generic type, this event explicitly defines itself as a one-way notification. Systems that subscribe to this event are expected to react to it but not to return a value or modify the outcome.

## Lifecycle & Ownership
- **Creation:** This event is instantiated exclusively by the server's core network layer, typically within the PacketHandler or a related I/O processing component. It is created at the precise moment a client's TCP connection is severed or times out during the pre-game login sequence.

- **Scope:** The object's lifetime is exceptionally brief and transient. It exists only for the duration of its dispatch through the server's central event bus.

- **Destruction:** Once all registered subscribers have processed the event, it becomes unreachable and is subsequently reclaimed by the Java Garbage Collector. It holds no persistent state and is not owned by any long-lived service.

## Internal State & Concurrency
- **State:** The PlayerSetupDisconnectEvent is **strictly immutable**. All its fields—username, uuid, auth, and disconnectReason—are declared as final and are populated only once at construction. This design guarantees that the event's data is a consistent snapshot of the connection state at the moment of failure.

- **Thread Safety:** Due to its immutability, this class is inherently thread-safe. It can be safely passed from a network I/O thread to the main server thread or any other worker thread without requiring locks or other synchronization primitives. This is a critical feature for high-performance, multi-threaded server architecture.

## API Surface
The public contract is limited to data accessors. There are no methods for mutation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUsername() | String | O(1) | Returns the username associated with the failed connection attempt. |
| getUuid() | UUID | O(1) | Returns the UUID of the player, if it was determined before the disconnect. |
| getAuth() | PlayerAuthentication | O(1) | Provides the authentication context object at the time of failure. |
| getDisconnectReason() | PacketHandler.DisconnectReason | O(1) | Returns an enum detailing the specific reason for the disconnection. |

## Integration Patterns

### Standard Usage
This event is not intended to be created or called directly. Instead, core server systems subscribe to it to perform cleanup or logging operations related to failed logins.

```java
// Example of a listener within a SessionManagementService
@Subscribe
public void onPlayerSetupDisconnect(PlayerSetupDisconnectEvent event) {
    // A player failed to connect fully. Clean up any temporary resources.
    log.warn("Player {} ({}) disconnected during setup. Reason: {}", 
        event.getUsername(), 
        event.getUuid(), 
        event.getDisconnectReason());
        
    authenticationCache.invalidate(event.getUuid());
    pendingSessions.remove(event.getUuid());
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Never create an instance of PlayerSetupDisconnectEvent in game logic. Firing this event manually can desynchronize the server's understanding of connected clients, leading to resource leaks or inconsistent state. It is exclusively an internal event from the network layer.

- **Misinterpretation:** Do not handle this event as a substitute for a standard PlayerDisconnectEvent. This event signifies a pre-join failure; no player entity was spawned, and no game state was associated with the connection. Treating it as a full player departure will cause errors.

- **State Assumption:** Do not assume all fields are non-null. While the username and reason are typically present, the UUID and auth object may be null if the disconnect occurred very early in the handshake process. Always perform null checks.

## Data Pipeline
The PlayerSetupDisconnectEvent represents a specific point in the server's data flow where the standard player connection pipeline is aborted.

> Flow:
> Client Connection Attempt -> TCP Handshake -> Server Network Layer -> **PlayerSetupDisconnectEvent Fired** -> Server Event Bus -> System Listeners (Logging, Auth Cache Cleanup)

