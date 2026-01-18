---
description: Architectural reference for the PlayerStorage interface, the core contract for player data persistence.
---

# PlayerStorage

**Package:** com.hypixel.hytale.server.core.universe.playerdata
**Type:** Interface / Service Contract

## Definition
```java
// Signature
public interface PlayerStorage {
    @Nonnull
    CompletableFuture<Holder<EntityStore>> load(@Nonnull UUID var1);

    @Nonnull
    CompletableFuture<Void> save(@Nonnull UUID var1, @Nonnull Holder<EntityStore> var2);

    @Nonnull
    CompletableFuture<Void> remove(@Nonnull UUID var1);

    @Nonnull
    Set<UUID> getPlayers() throws IOException;
}
```

## Architecture & Concepts
The PlayerStorage interface defines the abstract contract for all player data persistence operations on the server. It serves as a critical abstraction layer, decoupling the server's core universe logic from the underlying physical storage medium, which could be a local file system, a remote database, or a cloud object store.

This component's primary responsibility is to manage the serialization and deserialization of a player's state, encapsulated within an EntityStore. By mandating the use of CompletableFuture for its core I/O operations (load, save, remove), the interface enforces an asynchronous, non-blocking design. This is essential for server performance, preventing I/O-bound tasks from stalling the main game loop.

Implementations of this interface are expected to be robust, thread-safe, and handle all aspects of data integrity and I/O error management.

### Lifecycle & Ownership
As an interface, PlayerStorage itself has no lifecycle. The following describes the expected lifecycle of a concrete implementation (e.g., a FilePlayerStorage or DatabasePlayerStorage).

- **Creation:** A single instance of a PlayerStorage implementation is instantiated by the server's dependency injection or service management framework during the initial bootstrap sequence. It is a foundational service required for the server to function.
- **Scope:** The instance is a singleton that persists for the entire lifetime of the server process. It holds and manages resources like database connections or file system handles.
- **Destruction:** The implementation is expected to release any held resources (e.g., close database connection pools) during the server shutdown sequence.

## Internal State & Concurrency
- **State:** The interface is stateless. However, any concrete implementation will be inherently stateful. It may maintain a connection pool to a database, cache recently accessed player data in memory to reduce I/O latency, or manage file locks to prevent data corruption.

- **Thread Safety:** **CRITICAL:** Implementations of this interface **must be thread-safe**. The server will invoke its methods from multiple threads concurrently as players join and leave the game. The asynchronous nature of the API facilitates this, as I/O work is offloaded to a dedicated thread pool. Implementations must use appropriate concurrency controls (e.g., locks, concurrent collections) to protect internal state and ensure atomic operations on the underlying storage.

## API Surface
The public contract is designed for asynchronous, high-performance data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(UUID) | CompletableFuture | I/O Bound | Asynchronously loads a player's data from persistence. The future completes with the player's EntityStore or fails with an exception if the data is not found or corrupt. |
| save(UUID, Holder) | CompletableFuture | I/O Bound | Asynchronously saves a player's EntityStore to persistence. The future completes when the write operation is confirmed. Failure to handle exceptions on this future can lead to silent data loss. |
| remove(UUID) | CompletableFuture | I/O Bound | Asynchronously deletes a player's data. This is a destructive operation and should be used with caution, for example, during a character deletion process. |
| getPlayers() | Set | I/O Bound | **WARNING:** This is a potentially blocking operation that retrieves all known player UUIDs from the storage backend. It can cause performance degradation if the player dataset is large. It should not be called on the main game thread. |

## Integration Patterns

### Standard Usage
The PlayerStorage service is typically injected into a higher-level manager, such as a PlayerSessionManager or Universe, which orchestrates the player lifecycle.

```java
// Correct asynchronous handling of a player joining
public void onPlayerConnect(UUID playerUUID) {
    PlayerStorage storage = serverContext.getService(PlayerStorage.class);

    storage.load(playerUUID)
        .thenAcceptAsync(entityStoreHolder -> {
            // Logic to spawn the player in the world with their loaded data
            world.spawnPlayer(playerUUID, entityStoreHolder.get());
        }, mainGameThreadExecutor)
        .exceptionally(error -> {
            // Handle failure to load player data (e.g., kick player, log error)
            log.error("Failed to load player data for " + playerUUID, error);
            return null;
        });
}
```

### Anti-Patterns (Do NOT do this)
- **Blocking the Main Thread:** Never call .get() or .join() on the returned CompletableFuture from the main server thread. This will freeze the server until the I/O operation completes, causing catastrophic performance issues.

    ```java
    // DO NOT DO THIS
    // This blocks the entire server tick loop.
    EntityStore store = storage.load(uuid).get();
    ```

- **Ignoring Future Exceptions:** Failing to attach an .exceptionally() or .handle() block to the future chain means that any I/O errors during load or save operations will be silently ignored, potentially leading to player data loss or corruption.

- **Frequent getPlayers() Calls:** Calling getPlayers() repeatedly, especially on a critical path, is a significant performance hazard. The result should be cached if it is needed frequently.

## Data Pipeline
The PlayerStorage interface acts as the gatekeeper between in-memory game state and on-disk (or in-database) persistent state.

> **Load Flow:**
> Player Connect Event → PlayerSessionManager → **PlayerStorage.load()** → I/O Worker Thread (Reads from Disk/DB) → Data Deserialization → CompletableFuture Completion → Game Thread (Consumes EntityStore)

> **Save Flow:**
> Player Disconnect Event → PlayerSessionManager → **PlayerStorage.save()** → Data Serialization → I/O Worker Thread (Writes to Disk/DB) → CompletableFuture Completion → Post-Save Logic (if any)

