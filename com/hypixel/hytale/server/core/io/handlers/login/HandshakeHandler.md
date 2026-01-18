---
description: Architectural reference for HandshakeHandler
---

# HandshakeHandler

**Package:** com.hypixel.hytale.server.core.io.handlers.login
**Type:** Transient State Machine

## Definition
```java
// Signature
public abstract class HandshakeHandler extends GenericConnectionPacketHandler {
```

## Architecture & Concepts
The HandshakeHandler is a critical security component that orchestrates the entire player authentication sequence. It functions as a short-lived, stateful controller for a new client connection, guiding it through a multi-step mutual authentication process before allowing entry into the game world.

This handler is not a simple packet processor; it is a sophisticated state machine, explicitly governed by the internal AuthState enum. Its primary responsibility is to mediate a three-party security dance between the connecting Hytale Client, the Hytale Game Server, and Hytale's central Session Service.

The core architectural goals of this system are:
1.  **Client Identity Verification:** To cryptographically verify that the connecting player is who they claim to be, using JWT identity tokens.
2.  **Server Identity Verification (Mutual Authentication):** To prove to the client that the server is a legitimate, registered game server. This prevents man-in-the-middle attacks.
3.  **Session Establishment:** To securely transition the connection from an unauthenticated state to a fully authenticated game session.

Upon successful completion, the HandshakeHandler replaces itself in the network pipeline with the next-stage handler, effectively "graduating" the connection to an active player session.

### Lifecycle & Ownership
The lifecycle of a HandshakeHandler is ephemeral and tightly coupled to the authentication phase of a single network channel.

-   **Creation:** A HandshakeHandler is instantiated by a preceding network handler (e.g., a LoginDecoder) immediately after a client provides its initial connection request and identity token. It is not managed by a dependency injection container and should never be created manually.
-   **Scope:** The handler exists only for the duration of the authentication handshake, which is typically measured in seconds. Its state is relevant only to the single connection it manages.
-   **Destruction:** The object is dereferenced and becomes eligible for garbage collection under two conditions:
    1.  **Success:** After successfully authenticating the player, it calls the abstract `onAuthenticated` method, which is responsible for replacing this handler in the Netty pipeline.
    2.  **Failure:** If any step of the authentication process fails, times out, or a protocol error occurs, the handler terminates the connection, and it is subsequently destroyed.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. Its behavior is dictated by the volatile `authState` field, which transitions through several stages (e.g., `REQUESTING_AUTH_GRANT`, `AWAITING_AUTH_TOKEN`). It also caches player identity details and tracks received packets to prevent replay attacks.

-   **Thread Safety:** **This class is not thread-safe and must be considered thread-hostile.** All interactions with a HandshakeHandler instance must be performed on the Netty EventLoop thread assigned to its specific channel. Asynchronous operations, such as calls to the SessionServiceClient, are carefully marshaled back onto the correct EventLoop thread using `channel.eventLoop().execute()`.

    **WARNING:** Directly invoking methods on this handler from an external thread will lead to race conditions, state corruption, and unpredictable connection failures.

## API Surface
The public contract is primarily defined by its role as a `PacketHandler` in the Netty pipeline.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registered0(oldHandler) | void | O(N) | Lifecycle callback. Initiates the entire authentication flow, including JWT validation and the first call to the session service. Complexity is network-dependent. |
| accept(packet) | void | O(1) | Routes incoming packets to the appropriate `handle` method based on the current authentication state. |
| onAuthenticated(challenge) | void | N/A | Abstract callback triggered upon successful completion of the entire handshake. Subclasses must implement this to transition the player to the game world. |

## Integration Patterns

### Standard Usage
This class is designed to be extended, not directly invoked. A concrete implementation provides the final logic for transitioning a successfully authenticated player into the game.

```java
// Example of a concrete implementation
public class GameHandshakeHandler extends HandshakeHandler {
    // Constructor matching the parent...

    @Override
    protected void onAuthenticated(byte[] passwordChallenge) {
        // This method is the exit point of the handshake.
        // The connection is now considered secure and authenticated.

        // 1. Create the final PlayerConnection object.
        PlayerAuthentication authDetails = this.getAuthentication();
        PlayerConnection player = new PlayerConnection(this.channel, authDetails);

        // 2. Create the handler for the main game loop.
        GameplayHandler gameplayHandler = new GameplayHandler(player);

        // 3. Atomically replace this handler in the pipeline to transition
        //    the connection to the gameplay phase.
        this.channel.pipeline().replace(this, "gameplay-handler", gameplayHandler);

        LOGGER.info("Player %s has successfully authenticated and entered the game.", authDetails.getUsername());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new HandshakeHandler()`. The server's network layer is responsible for creating it and inserting it into the channel pipeline at the correct time.
-   **State Manipulation:** Do not attempt to modify the `authState` or other internal fields directly. The handler's internal state machine manages this automatically.
-   **Multi-threaded Access:** Never share a HandshakeHandler instance across threads or call its methods from outside the channel's assigned EventLoop. This will break the concurrency model.
-   **Reusing Instances:** A HandshakeHandler is single-use. It cannot be reused for another connection or a reconnection attempt.

## Data Pipeline
The HandshakeHandler orchestrates a complex flow of data between the client, server, and external authentication services.

> Flow:
> Client Identity Token -> **HandshakeHandler** -> (HTTPS) Hytale Session Service (Validation & Grant Request) -> AuthGrant Packet -> Client -> AuthToken Packet -> **HandshakeHandler** -> (HTTPS) Hytale Session Service (Mutual Auth Exchange) -> ServerAuthToken Packet -> Authenticated Session & Gameplay Handler

