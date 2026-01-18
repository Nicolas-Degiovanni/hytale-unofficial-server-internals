---
description: Architectural reference for PlayerAuthentication
---

# PlayerAuthentication

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient Data Object

## Definition
```java
// Signature
public class PlayerAuthentication {
```

## Architecture & Concepts
PlayerAuthentication is a fundamental data transfer object (DTO) that encapsulates a player's identity during the server connection and authentication sequence. It is not a service or manager; it is a short-lived container for critical identity credentials.

This class acts as a stateful carrier, passed between various stages of the server's authentication pipeline. Its design, featuring a default constructor and individual setters, allows for the incremental assembly of a player's identity as it is verified. For example, an initial network handler might create the object, a subsequent service call populates the UUID and username, and a final system consumes the completed object to spawn the player in the world.

The inclusion of referral data and source indicates its role extends to tracking player acquisition channels, which is critical for server analytics and multi-server proxy environments. The non-null enforcement in core getters acts as a safety mechanism, ensuring that downstream systems cannot operate on a partially authenticated or incomplete identity.

## Lifecycle & Ownership
- **Creation:** A new PlayerAuthentication instance is created by a low-level network or authentication handler at the very beginning of a player's connection attempt. The default `PlayerAuthentication()` constructor is typically invoked.
- **Scope:** The object's lifetime is strictly bound to a single connection attempt. It persists only for the duration of the login and handshake process.
- **Destruction:** Once the authentication process is complete, the data within this object is typically transferred to a more permanent, long-lived entity like a Player or Session object. The PlayerAuthentication instance is then discarded and becomes eligible for garbage collection. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable**. All fields are exposed via public setters, which is intentional to support the progressive build-up of the authentication context. This object is a temporary state machine for a single login flow.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization primitives such as locks or atomic variables.

    **WARNING:** All operations on a PlayerAuthentication instance must be confined to the single thread responsible for processing a specific player's connection. Sharing this object across multiple threads without external locking will result in severe data corruption and unpredictable authentication failures.

## API Surface
The public API is designed for populating and then reading a completed authentication record.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUsername() | String | O(1) | Retrieves the validated username. Throws UnsupportedOperationException if the username has not been set. |
| getUuid() | UUID | O(1) | Retrieves the validated player UUID. Throws UnsupportedOperationException if the UUID has not been set. |
| setReferralData(data) | void | O(1) | Sets optional referral data. Throws IllegalArgumentException if data exceeds 4096 bytes. |

## Integration Patterns

### Standard Usage
The object should be instantiated early in the connection pipeline and populated sequentially as identity information is verified.

```java
// In an authentication handler for a new network connection
PlayerAuthentication authContext = new PlayerAuthentication();

// After validating credentials against a master authentication service
authContext.setUuid(validatedPlayerUUID);
authContext.setUsername(validatedPlayerUsername);

// Pass the completed context to the next system
playerSessionFactory.createSession(authContext);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and re-use PlayerAuthentication objects between different player sessions. This will leak sensitive state and cause severe identity bugs. Always create a new instance for each connection attempt.
- **Premature Access:** Do not pass the object to core game logic systems before both the UUID and username have been successfully populated. Downstream systems rely on the contract that `getUsername()` and `getUuid()` will succeed.
- **Cross-thread Modification:** Never modify a PlayerAuthentication object from a different thread than the one that owns the player's connection pipeline.

## Data Pipeline
PlayerAuthentication serves as the primary data vessel carrying identity from the network edge into the core server logic.

> Flow:
> Network Handshake -> Authentication Service Client -> **PlayerAuthentication** (Creation & Population) -> Session Manager -> Player Entity Creation

