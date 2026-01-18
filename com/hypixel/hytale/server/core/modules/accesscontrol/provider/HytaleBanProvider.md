---
description: Architectural reference for HytaleBanProvider
---

# HytaleBanProvider

**Package:** com.hypixel.hytale.server.core.modules.accesscontrol.provider
**Type:** Singleton Service

## Definition
```java
// Signature
public class HytaleBanProvider extends BlockingDiskFile implements AccessProvider {
```

## Architecture & Concepts
The HytaleBanProvider is a concrete implementation of the AccessProvider interface, serving as the primary persistence and caching layer for all player bans on the server. Its core responsibility is to manage the lifecycle of ban data, bridging the gap between the on-disk representation (bans.json) and a fast, in-memory cache used for runtime access checks.

By extending BlockingDiskFile, it inherits a robust, lock-based mechanism for managing file I/O, ensuring that all disk operations are synchronized and safe from race conditions. The provider maintains an in-memory map of active bans, which is hydrated from bans.json upon server startup. This design ensures that player connection requests can be checked against the ban list with minimal latency, as lookups only query the in-memory map and do not incur disk I/O overhead during normal operation.

All modifications to the ban list are funneled through the `modify` method, which guarantees atomic updates to the in-memory cache and triggers an asynchronous write-through to the bans.json file, ensuring data durability.

### Lifecycle & Ownership
- **Creation:** Instantiated and managed by the AccessControlModule during the server's bootstrap sequence. It is not intended for direct instantiation by other services.
- **Scope:** The HytaleBanProvider is a long-lived service. Its lifecycle is tied directly to the server session, persisting from server start to shutdown.
- **Destruction:** The object is eligible for garbage collection upon server shutdown when the AccessControlModule is dismantled. There is no explicit destruction or cleanup method; file handles are managed by the parent BlockingDiskFile.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. The provider holds an in-memory `bans` map of type Object2ObjectOpenHashMap, which serves as a write-through cache for the contents of bans.json. This map is populated at startup and modified at runtime when bans are added or removed.
- **Thread Safety:** This class is designed to be thread-safe, primarily through the locking mechanisms inherited from BlockingDiskFile.
    - **Write Operations:** The `modify` method acquires a full write lock, providing exclusive access to the internal `bans` map and preventing any concurrent reads or writes.
    - **Read Operations:** The `hasBan` method correctly uses a read lock, allowing for concurrent, non-blocking reads.
    - **WARNING:** The `getDisconnectReason` method modifies the internal map by removing expired bans but does so without acquiring a write lock. This presents a potential concurrency risk. A call to this method during a separate thread's iteration over the map could result in a ConcurrentModificationException.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDisconnectReason(UUID uuid) | CompletableFuture<Optional<String>> | O(1) | Asynchronously retrieves the disconnect reason for a banned player. Returns an empty Optional if the player is not banned. Lazily removes expired bans it encounters. |
| hasBan(UUID uuid) | boolean | O(1) | Performs a fast, thread-safe, read-only check to determine if a player is in the ban cache. |
| modify(Function<Map<UUID, Ban>, Boolean> function) | boolean | O(F) | Atomically executes a mutation on the internal ban map. F is the complexity of the supplied function. If the function returns true, changes are persisted to disk. This is the **only** supported method for altering ban state. |

## Integration Patterns

### Standard Usage
The provider must be retrieved from the server's central AccessControlModule. Modifications should always be performed via the `modify` method, passing a lambda to ensure atomicity and proper persistence.

```java
// Correctly adding a new ban
AccessControlModule acm = server.getModuleManager().getModule(AccessControlModule.class);
HytaleBanProvider banProvider = acm.getBanProvider();

UUID targetPlayer = ...;
Ban newBan = Ban.newBuilder().target(targetPlayer).reason("Exploiting glitches").build();

boolean wasModified = banProvider.modify(bans -> {
    bans.put(targetPlayer, newBan);
    return true; // Signal that a change was made
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new HytaleBanProvider()`. This will result in a disconnected provider managing a separate, out-of-sync bans.json file, leading to inconsistent server state and bypassing the module system's lifecycle management.
- **Unsafe Iteration:** Do not retrieve and hold a reference to the internal map for external iteration. The `getDisconnectReason` method can cause concurrent modification exceptions if it removes an expired ban while another thread is iterating. All operations must be encapsulated within the `modify` function.
- **Bypassing Persistence:** If the function passed to `modify` returns false, no changes will be saved to disk, even if the map was altered. Always return true after a successful mutation.

## Data Pipeline
The flow of ban data is managed through distinct read, check, and write pipelines.

> **Read Pipeline (Server Startup):**
> `bans.json` on Disk → `BlockingDiskFile` I/O → `HytaleBanProvider.read()` → GSON Deserialization → In-Memory `bans` Map

> **Check Pipeline (Player Login):**
> Player UUID → `AccessControlModule.getDisconnectReason()` → `HytaleBanProvider.getDisconnectReason()` → In-Memory `bans` Map Lookup → Optional Disconnect Reason

> **Write Pipeline (Ban Command):**
> Server Command → `HytaleBanProvider.modify()` → In-Memory `bans` Map Mutation → `BlockingDiskFile.syncSave()` → `HytaleBanProvider.write()` → GSON Serialization → `bans.json` on Disk

