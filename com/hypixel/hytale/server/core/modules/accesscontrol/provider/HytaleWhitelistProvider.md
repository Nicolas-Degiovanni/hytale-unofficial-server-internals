---
description: Architectural reference for HytaleWhitelistProvider
---

# HytaleWhitelistProvider

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.provider
**Type:** Stateful Service

## Definition
```java
// Signature
public class HytaleWhitelistProvider extends BlockingDiskFile implements AccessProvider {
```

## Architecture & Concepts
The HytaleWhitelistProvider is a concrete implementation of the AccessProvider strategy. It serves as a gatekeeper during the player connection lifecycle, enforcing a server whitelist based on player UUIDs.

Architecturally, this class is a self-contained component that bridges server security policy with persistent storage. By extending BlockingDiskFile, it tightly couples its in-memory state with a corresponding `whitelist.json` file on disk. This design simplifies state management by delegating file I/O mechanics to its parent class, allowing HytaleWhitelistProvider to focus solely on the logic of whitelist management and JSON serialization.

Its primary role within the server is to provide a fast, thread-safe answer to the question: "Is this player allowed to join?". It maintains an in-memory `HashSet` of whitelisted UUIDs for O(1) average-time lookups, ensuring that the access control check imposes minimal latency on the connection handshake process.

**WARNING:** The reliance on BlockingDiskFile means that all persistence operations (saving the whitelist) are synchronous and will block the calling thread. While modifications are expected to be infrequent, this can introduce latency if the disk is under heavy load.

## Lifecycle & Ownership
- **Creation:** Instantiated once during the server's bootstrap sequence, typically by a parent `AccessControlModule` or a central service registry. Upon instantiation, the parent BlockingDiskFile constructor is invoked, which may trigger an initial, blocking read from `whitelist.json` to populate the in-memory state.
- **Scope:** This object is a long-lived service. Its lifecycle is bound to the server process itself, persisting from server start to server stop.
- **Destruction:** The object is eligible for garbage collection upon server shutdown. No explicit cleanup methods are exposed; file handles are managed by the parent class and the underlying Java runtime.

## Internal State & Concurrency
- **State:** The HytaleWhitelistProvider is stateful and highly mutable. It maintains two core pieces of state:
    1.  A boolean `isEnabled` flag that globally toggles the whitelist check.
    2.  A `Set<UUID>` named `whitelist` that serves as an in-memory cache of the `whitelist.json` file content.
- **Thread Safety:** This class is explicitly designed to be thread-safe. All access and mutation of its internal state are guarded by a `ReentrantReadWriteLock`. This lock strategy is optimized for a "read-mostly" workload, where many players may be connecting simultaneously (requiring read access) while whitelist modifications (requiring write access) are rare. This allows for high-throughput, non-blocking reads while ensuring that writes are atomic and isolated.

## API Surface
The public contract is designed for two distinct use cases: checking access and modifying the list.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisconnectReason(UUID uuid) | CompletableFuture<Optional<String>> | O(1) | The primary integration point for the connection pipeline. Returns an empty Optional if access is granted, or a reason string if denied. |
| modify(Function<Set<UUID>, Boolean> consumer) | boolean | O(N) | Atomically modifies the whitelist set. The provided function receives a mutable reference to the set. If the function returns true, a blocking save to disk is triggered. |
| setEnabled(boolean isEnabled) | void | O(1) | Enables or disables the whitelist in-memory. **WARNING:** This method does not persist the change to disk. The state will be lost on restart unless a subsequent `modify` call triggers a save. |
| getList() | Set<UUID> | O(N) | Returns a thread-safe, *unmodifiable* snapshot of the current whitelist. |
| isEnabled() | boolean | O(1) | Returns the current in-memory status of the whitelist. |

## Integration Patterns

### Standard Usage
Modification of the whitelist should always be performed via the `modify` method to ensure atomicity and persistence. This is the expected pattern for server commands like `/whitelist add` or `/whitelist remove`.

```java
// How a server command should add a player to the whitelist
HytaleWhitelistProvider provider = server.getAccessControlModule().getWhitelistProvider();
UUID playerToAdd = ...;

boolean wasAdded = provider.modify(whitelist -> {
    return whitelist.add(playerToAdd);
});

if (wasAdded) {
    // The change is now live and has been saved to whitelist.json
    System.out.println("Player added to whitelist.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new HytaleWhitelistProvider()`. The server relies on a single, shared instance to manage the whitelist state. Always retrieve it from the appropriate service registry or module.
- **Unsaved State Changes:** Do not call `setEnabled(boolean)` and assume the change is permanent. This only affects the in-memory state. To persist a change to the enabled flag, it must be done within the context of a `write` operation, which is triggered by `modify`.
- **Modifying a Retrieved List:** The set returned by `getList()` is unmodifiable. Attempting to call `add()` or `remove()` on it will result in an `UnsupportedOperationException`.

## Data Pipeline
The component has two primary data flows: the access check flow and the modification flow. The access check flow is critical to the player connection process.

> **Access Check Flow:**
> Player Connection -> Server Handshake Logic -> **HytaleWhitelistProvider.getDisconnectReason()** -> In-Memory HashSet Lookup -> Access Decision -> Allow/Deny Connection

> **Modification Flow:**
> Admin Command -> Command Handler -> **HytaleWhitelistProvider.modify()** -> In-Memory HashSet Mutation -> **BlockingDiskFile.syncSave()** -> JSON Serialization -> `whitelist.json` on Disk

