---
description: Architectural reference for AccessProvider
---

# AccessProvider

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.provider
**Type:** Service Contract / Interface

## Definition
```java
// Signature
public interface AccessProvider {
```

## Architecture & Concepts
The AccessProvider interface defines a fundamental contract within the server's security and connection-handling pipeline. It serves as an asynchronous gatekeeper, responsible for determining whether a connecting player should be granted access to the server.

This component is invoked early in the player connection lifecycle. Its primary architectural purpose is to decouple the connection logic from the specific rules of access control. By using an asynchronous pattern with CompletableFuture, implementations can perform potentially long-running operations—such as database queries, external API calls to a global ban service, or complex permission checks—without blocking the server's primary network threads. This is critical for maintaining server responsiveness and scalability under high connection loads.

The result is communicated via an Optional String. An empty Optional signifies that access is granted by this provider, allowing the connection process to continue. A populated Optional contains a non-null String, which is used as the disconnection reason sent to the client, immediately terminating the connection attempt.

## Lifecycle & Ownership
As an interface, AccessProvider itself does not have a lifecycle. The following applies to its concrete implementations.

- **Creation:** Implementations are expected to be instantiated by the server's dependency injection or service management framework during the server bootstrap sequence. They are typically registered as services.
- **Scope:** An AccessProvider implementation is a long-lived service. Its lifetime is tied to the server session, persisting from server start to shutdown.
- **Destruction:** The provider is destroyed and its resources are released during the server shutdown process.

## Internal State & Concurrency
- **State:** The interface is stateless. Implementations, however, may be stateful. For example, a provider might cache recent access decisions to reduce latency and load on backend systems. Any such state must be managed carefully.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. The getDisconnectReason method will be invoked concurrently from multiple network worker threads as players attempt to connect simultaneously. All internal state, such as caches or connection pools, must be protected against race conditions using appropriate concurrency controls.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisconnectReason(UUID) | CompletableFuture<Optional<String>> | Asynchronous | Asynchronously checks if the player identified by the UUID is permitted to join. Returns a future that completes with an empty Optional on success, or an Optional containing the kick reason on failure. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a central service registry and integrated into the player login handler. The asynchronous result is handled using callbacks to avoid blocking the network thread.

```java
// In a connection handler...
AccessProvider provider = serviceRegistry.get(AccessProvider.class);
UUID playerUUID = incomingConnection.getUUID();

provider.getDisconnectReason(playerUUID).thenAccept(reasonOptional -> {
    if (reasonOptional.isPresent()) {
        // Deny access and disconnect the player
        playerConnection.disconnect(reasonOptional.get());
    } else {
        // Access granted, proceed with the next login step
        continueLoginProcess(playerConnection);
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Blocking on the Future:** Never call get() on the returned CompletableFuture from a network I/O thread or the main server tick loop. This will freeze the thread, severely degrading or crashing the server. Always use asynchronous callbacks like thenAccept or whenComplete.
- **Ignoring the Future:** Failing to handle the result of the future will lead to players being stuck in a pending login state. Every call must be handled.
- **Swallowing Exceptions:** Implementations should properly handle exceptions within the CompletableFuture by completing it exceptionally. Consumers of the API should use exceptionally() to handle failed checks gracefully, rather than letting the exception propagate up the stack.

## Data Pipeline
The AccessProvider is a critical decision point in the player connection data flow.

> Flow:
> Player Connection Request -> Network Layer -> **AccessProvider.getDisconnectReason** -> [Asynchronous Check: DB/API/Cache] -> Future Completion -> Connection Handler -> [Continue Login | Send Disconnect Packet]

