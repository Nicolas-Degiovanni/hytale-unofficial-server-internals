---
description: Architectural reference for PlayerSetupConnectEvent
---

# PlayerSetupConnectEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient Event Object

## Definition
```java
// Signature
public class PlayerSetupConnectEvent implements IEvent<Void>, ICancellable {
```

## Architecture & Concepts
The PlayerSetupConnectEvent is a critical control point in the server's player connection pipeline. It is dispatched *after* a player's identity has been successfully authenticated but *before* they are fully admitted into the game world.

This event acts as a mutable context object, allowing server logic, such as plugins or custom game modes, to intercept and modify the outcome of a connection attempt. Its primary architectural purpose is to decouple connection validation and routing logic from the core networking stack. By listening for this event, developers can implement a wide range of features without modifying the server's internal connection handler.

Common use cases include:
*   Implementing server whitelists or dynamic player slot management.
*   Performing complex, secondary authentication or data validation.
*   Redirecting players to different servers, forming the basis of a lobby or hub-and-spoke server architecture.
*   Denying connections based on custom logic (e.g., maintenance mode, region locks).

The event carries all necessary state for making these decisions, including the player's identity, authentication details, and the low-level network handler.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's core network pipeline. This occurs precisely after the initial cryptographic and identity handshake succeeds but before the client is transitioned to the "in-game" state.
- **Scope:** Ephemeral and extremely short-lived. The object's lifetime is strictly confined to a single, synchronous dispatch cycle on the server's event bus. The entire connection process is blocked until all listeners have processed this event.
- **Destruction:** The object becomes eligible for garbage collection immediately after the event dispatch completes. Listeners **must not** retain references to the event object after their handler method returns.

## Internal State & Concurrency
- **State:** Highly mutable by design. While initial player and connection details are immutable, the event's outcome fields (cancelled, reason, clientReferral) are intended to be modified by listeners. The final state of these fields after the event dispatch determines the connection's fate.
- **Thread Safety:** **Not thread-safe.** This event is designed to be processed synchronously by all listeners on a single server thread. Concurrent modification from multiple threads will result in race conditions and undefined behavior.

**WARNING:** Do not attempt to process this event asynchronously. The connection pipeline depends on the immediate, synchronous result of the event dispatch. Offloading logic to another thread will break the connection flow.

## API Surface
The public contract is focused on inspecting the connection attempt and modifying its outcome.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. If true, the player connection will be terminated using the provided reason. |
| setReason(String) | void | O(1) | Sets the disconnection message sent to the player if the event is cancelled. Throws NullPointerException if the reason is null. |
| referToServer(host, port, data) | void | O(1) | Configures a server referral. This cancels the current connection and instructs the client to connect to the new address. Throws IllegalArgumentException for invalid ports or oversized data. |
| isReferralConnection() | boolean | O(1) | Returns true if the player is connecting via a referral from another server. |
| getPacketHandler() | PacketHandler | O(1) | Provides access to the low-level network handler for this connection. For advanced use only. |
| getAuth() | PlayerAuthentication | O(1) | Returns the authentication details for the connecting player. |

## Integration Patterns

### Standard Usage
The standard pattern is to create an event listener that inspects the event state and conditionally cancels it or refers the player.

```java
// Example: A listener that enforces a maintenance mode
public void onPlayerSetupConnect(PlayerSetupConnectEvent event) {
    if (ServerConfig.isMaintenanceMode() && !event.getAuth().isOperator()) {
        event.setCancelled(true);
        event.setReason("The server is currently in maintenance mode.");
    }
}

// Example: A listener that redirects players to a lobby server
public void routeToLobby(PlayerSetupConnectEvent event) {
    if (GameWorld.isFull()) {
        event.referToServer("lobby.hytale.com", 25565);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PlayerSetupConnectEvent()`. This object is exclusively created and dispatched by the server's core. Manually creating one will have no effect on any player connection.
- **Asynchronous Handling:** Do not pass the event object to another thread for processing. The connection is paused awaiting the synchronous completion of the event dispatch. Delaying its completion will stall the connection pipeline.
- **Holding References:** Do not store the event object in a field or collection. It is a transient object that is invalid and subject to garbage collection as soon as the event dispatch is finished.

## Data Pipeline
This event serves as a gatekeeper in the data flow of a new player connection.

> Flow:
> Client Handshake -> Server Network Layer -> Authentication Service -> **PlayerSetupConnectEvent** -> Event Bus -> (Listener Chain) -> Connection Logic -> Player Admitted **OR** Disconnect Packet Sent **OR** Referral Packet Sent

