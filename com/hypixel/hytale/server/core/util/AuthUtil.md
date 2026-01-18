---
description: Architectural reference for AuthUtil
---

# AuthUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class AuthUtil {
```

## Architecture & Concepts
The AuthUtil class is a stateless utility component responsible for resolving a player's username into a universally unique identifier (UUID). Architecturally, it serves as a compatibility layer for an offline or development environment where a central, external authentication authority is not present.

Its primary function is to query the active game state, represented by the Universe singleton, for a player matching the given username. If no active player is found, it falls back to a deterministic UUID generation scheme. This fallback mechanism is critical for server stability, ensuring that systems requiring a UUID can still function even for players who are not yet fully registered in the game world.

**Warning:** The entire class is built around a non-authenticated model. The UUIDs it produces, especially in the fallback case, carry no security guarantees and should not be used for session validation or permissions enforcement against external systems. The use of CompletableFuture in its API suggests a design that was once intended to be asynchronous (e.g., making a network call to an authentication server), but the current implementation is fully synchronous and blocking.

## Lifecycle & Ownership
- **Creation:** As a static utility class, AuthUtil is never instantiated. Its bytecode is loaded by the JVM's classloader when first referenced.
- **Scope:** The class and its static methods are available for the entire lifetime of the server application.
- **Destruction:** The class is unloaded when the server process terminates and its classloader is garbage collected.

## Internal State & Concurrency
- **State:** AuthUtil is completely stateless. It does not contain any member variables, caches, or configuration. Each method call operates exclusively on the arguments provided and the state of external systems like the Universe.
- **Thread Safety:** This class is inherently thread-safe. Since it holds no state, concurrent calls to its static methods will not interfere with each other. However, thread safety is dependent on the thread safety of the components it calls, specifically the Universe singleton. It is assumed that Universe provides safe concurrent access to its player data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| lookupUuid(String username) | CompletableFuture<UUID> | O(1) | **Deprecated.** Resolves a username to a UUID. First attempts a lookup in the live Universe. If not found, generates a deterministic offline-mode UUID. The operation is synchronous despite the CompletableFuture return type. |

## Integration Patterns

### Standard Usage
The following demonstrates the intended, albeit deprecated, use of this utility. It is typically used in legacy code during the early stages of player connection before a full Player object is established.

**Warning:** Do not use this method in new code. A modern authentication or player lookup service should be used instead.

```java
// This pattern is for reference of existing code only.
CompletableFuture<UUID> futureUuid = AuthUtil.lookupUuid("PlayerName");

futureUuid.thenAccept(uuid -> {
    // Logic to handle the resolved or generated UUID
    System.out.println("Resolved UUID: " + uuid);
});
```

### Anti-Patterns (Do NOT do this)
- **Trusting the UUID:** Do not use the UUID returned by this utility for any security-sensitive operations. It does not verify that the player is who they claim to be.
- **Assuming Asynchronicity:** Do not build complex asynchronous chains off the returned CompletableFuture. The current implementation completes immediately on the calling thread, and treating it as a long-running operation is misleading.
- **Use in New Features:** Do not call this method from any new feature development. Its deprecated status indicates it is scheduled for removal and is superseded by more robust player identification mechanisms.

## Data Pipeline
The data flow for this utility is a simple, synchronous lookup-or-generate sequence. It acts as a transformer from a string identifier to a UUID identifier.

> Flow:
> String username -> **AuthUtil.lookupUuid** -> Universe.getPlayerByUsername -> (PlayerRef Found?) -> Return PlayerRef.getUuid()
>
> Flow (Fallback):
> String username -> **AuthUtil.lookupUuid** -> Universe.getPlayerByUsername -> (PlayerRef Not Found) -> Generate Offline UUID -> Return Generated UUID

