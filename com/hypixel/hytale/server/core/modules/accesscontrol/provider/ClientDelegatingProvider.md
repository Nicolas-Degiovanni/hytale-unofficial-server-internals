---
description: Architectural reference for ClientDelegatingProvider
---

# ClientDelegatingProvider

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.provider
**Type:** Transient

## Definition
```java
// Signature
public class ClientDelegatingProvider implements AccessProvider {
```

## Architecture & Concepts
The ClientDelegatingProvider is a concrete implementation of the AccessProvider strategy interface. Its primary architectural role is to serve as a default or "pass-through" mechanism within the server's access control framework.

Contrary to what its name might imply, this provider does not delegate the access check to the client. Instead, it represents a policy of **implicit trust**. It immediately and unconditionally approves any access request by returning an empty Optional, which signifies that there is no reason to disconnect the incoming connection.

This component is critical for scenarios where server-side access validation is disabled or not required, such as:
*   Local development environments.
*   Single-player or listen server instances.
*   Servers configured to operate in an offline or insecure mode.

It acts as a null object or a no-op implementation within the broader access control system, ensuring the system has a valid provider even when no active validation logic is configured.

### Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level service, typically the AccessControlModule or a related factory, during server bootstrap. The selection of this specific provider is driven by server configuration.
- **Scope:** The instance is stateless and lightweight. A single instance typically persists for the entire server session, shared across all connection-handling threads.
- **Destruction:** The object is eligible for garbage collection when the owning AccessControlModule is destroyed, which usually occurs during a graceful server shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no mutable or immutable fields and does not cache any data. Each invocation of its methods is independent and produces an identical outcome.
- **Thread Safety:** As a stateless object, ClientDelegatingProvider is inherently **thread-safe**. It can be safely invoked from multiple network or worker threads concurrently without requiring any external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisconnectReason(UUID uuid) | CompletableFuture<Optional<String>> | O(1) | Immediately returns a completed future containing an empty Optional. This method will never provide a disconnect reason, effectively approving all access requests. |

## Integration Patterns

### Standard Usage
This provider is not intended for direct invocation by application code. It is configured to be used internally by the server's primary access control service. A developer interacts with the abstraction, not the implementation.

```java
// The AccessControlService is configured to use ClientDelegatingProvider
// based on server settings. You would not instantiate it yourself.

AccessControlService accessControl = server.getService(AccessControlService.class);

// This call will internally route to ClientDelegatingProvider,
// which will always resolve to an empty Optional.
CompletableFuture<Optional<String>> result = accessControl.getDisconnectReason(playerUUID);

result.thenAccept(reason -> {
    if (!reason.isPresent()) {
        // This branch is always taken when ClientDelegatingProvider is active.
        System.out.println("Access granted.");
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Conditional Logic:** Do not write code that expects this provider to ever return a non-empty Optional. Logic that attempts to handle a disconnect reason from this specific provider is unreachable code.
- **Direct Instantiation:** Avoid creating an instance with `new ClientDelegatingProvider()`. The access control framework is responsible for its lifecycle. Manually creating it circumvents server configuration and can lead to unexpected behavior.

## Data Pipeline
The data flow for this component is minimal, acting as a terminal point in the decision-making process that always results in approval.

> Flow:
> Inbound Connection Request -> AccessControlService -> **ClientDelegatingProvider.getDisconnectReason()** -> CompletedFuture(Empty) -> Connection Approved

