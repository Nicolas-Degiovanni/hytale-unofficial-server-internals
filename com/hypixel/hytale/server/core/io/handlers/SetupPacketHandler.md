---
description: Architectural reference for SetupPacketHandler
---

# SetupPacketHandler

**Package:** com.hypixel.hytale.server.core.io.handlers
**Type:** Transient

## Definition
```java
// Signature
public class SetupPacketHandler extends GenericConnectionPacketHandler {
```

## Architecture & Concepts

The SetupPacketHandler is a critical, short-lived component that orchestrates the second stage of the player connection lifecycle. It acts as a state machine, guiding a client from a basic authenticated connection to a fully initialized presence within the game universe.

Its primary responsibility is to manage the pre-game handshake, which includes:
1.  **Server Configuration Exchange:** Sending essential server data like world settings, required assets, and server information to the client.
2.  **Asset Negotiation:** Receiving an asset request from the client and coordinating the transmission of required game assets and internationalization data.
3.  **Client Configuration Gathering:** Receiving the client's desired view radius and player options, including cosmetic skin data.
4.  **Player Instantiation:** Triggering the final, asynchronous creation of the player entity within the game's Universe.

This handler is the bridge between the low-level network pipeline and the high-level game simulation. Upon successful completion of its sequence, it is replaced by a permanent, in-game packet handler, and the SetupPacketHandler instance is discarded.

## Lifecycle & Ownership

-   **Creation:** Instantiated by the server's network layer immediately after a player's initial login and authentication is successful. It takes over the connection from a preceding handler, such as a login or handshake handler.
-   **Scope:** The handler's lifetime is strictly bound to the setup phase of a single client connection. It persists only for the few seconds required to negotiate assets and world settings.
-   **Destruction:** The handler is marked for garbage collection under three conditions:
    1.  **Success:** After successfully calling `Universe.addPlayer` and the returned `CompletableFuture` completes, the Netty channel pipeline is reconfigured with a new handler, releasing all references to this one.
    2.  **Failure:** If the client is disconnected for any reason (protocol error, timeout, server shutdown), the `closed` method is invoked and the handler is destroyed.
    3.  **Timeout:** The handler sets multiple internal timeouts. If the client fails to respond within the allotted time for a specific stage (e.g., sending `RequestAssets`), the connection is terminated, and the handler is destroyed.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. It maintains critical state for the connection handshake, including player identity (UUID, username), authentication status, and progress flags like `receivedRequest`. This internal state is essential for enforcing the correct sequence of packets and preventing protocol errors.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed by the Netty I/O thread assigned to its channel. All packet handling methods and lifecycle callbacks are guaranteed by Netty to execute serially on this event loop. Asynchronous operations, such as adding a player to the Universe, are managed via `CompletableFuture` to avoid blocking the I/O thread. Callbacks from these futures are marshaled back onto the event loop to ensure safe state mutation.

## API Surface

The public API consists of packet handling methods invoked by the network layer, not by general application code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registered0(oldHandler) | void | O(1) | Lifecycle hook. Initiates the setup sequence by sending WorldSettings and ServerInfo. Sets the initial timeout. |
| accept(packet) | void | O(1) | Main packet dispatcher. Routes incoming packets to the appropriate `handle` method based on packet ID. |
| handle(Disconnect) | void | O(1) | Handles a client-initiated disconnect packet. Logs the reason and closes the channel. |
| handle(RequestAssets) | void | O(N) | Handles the client's asset manifest. Triggers the asynchronous dispatch of common assets and translations. Throws an exception if called more than once. |
| handle(ViewRadius) | void | O(1) | Receives and stores the client's view radius for later use during player creation. |
| handle(PlayerOptions) | void | O(N) | Final step. Receives player skin and other options. Triggers the asynchronous `Universe.addPlayer` operation. |
| closed(ctx) | void | O(1) | Lifecycle hook. Dispatches a `PlayerSetupDisconnectEvent` and handles server shutdown logic in singleplayer mode. |

## Integration Patterns

### Standard Usage

Developers should not interact with this class directly. The primary integration point is through the Hytale Event Bus, listening for events dispatched by the handler.

```java
// Example of listening for the event that signals the start of the setup phase.
// This allows a plugin to modify the connection flow, for example, by redirecting the player.

@Subscribe
public void onPlayerSetup(PlayerSetupConnectEvent event) {
    // The event provides a reference to the handler, but direct manipulation is discouraged.
    String username = event.getUsername();
    
    if (isBanned(username)) {
        // Use the event's API to cancel the connection.
        event.setCancelled(true);
        event.setReason("You are banned from this server.");
    } else if (shouldRedirect(username)) {
        // Use the event's API to redirect the player to another server.
        ClientReferral referral = new ClientReferral("play.anotherserver.com", 25565);
        event.setClientReferral(referral);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new SetupPacketHandler()`. The server's internal connection manager is solely responsible for its lifecycle.
-   **State Tampering:** Do not access or modify the handler's internal state fields (e.g., `receivedRequest`) from outside the class, especially from other threads. This will break the connection state machine and cause unpredictable behavior.
-   **Blocking Operations:** Never perform blocking I/O or long-running computations within an event listener for `PlayerSetupConnectEvent`. Doing so will stall the Netty I/O thread, freezing a portion of the server.

## Data Pipeline

The SetupPacketHandler orchestrates a precise flow of data and control to transition a connection into an active player.

> Flow:
> Authenticated Netty Channel -> **SetupPacketHandler.registered0** -> `WorldSettings` Packet Sent -> Client Responds with `RequestAssets` Packet -> **SetupPacketHandler.handle(RequestAssets)** -> Server Sends Assets -> Client Responds with `PlayerOptions` Packet -> **SetupPacketHandler.handle(PlayerOptions)** -> `Universe.addPlayer` (async) -> New `PlayerPacketHandler` replaces `SetupPacketHandler` -> Player is in-game.

