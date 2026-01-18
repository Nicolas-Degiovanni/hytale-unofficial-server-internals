---
description: Architectural reference for DiskPlayerStorageProvider
---

# DiskPlayerStorageProvider

**Package:** com.hypixel.hytale.server.core.universe.playerdata
**Type:** Factory/Provider

## Definition
```java
// Signature
public class DiskPlayerStorageProvider implements PlayerStorageProvider {
```

## Architecture & Concepts
The DiskPlayerStorageProvider is a concrete implementation of the PlayerStorageProvider strategy interface. Its sole responsibility is to provide a mechanism for persisting and retrieving player data from the local filesystem. This component is a foundational piece of the server's persistence layer, abstracting the details of disk I/O from the core game logic that manages player state.

It operates by serializing a player's entity data, represented by an EntityStore, into a BSON format and writing it to a human-readable JSON file. This approach balances performance with debuggability. The file-per-player model, using the player's UUID as the filename, ensures simple, conflict-free storage without the need for a complex database.

The presence of a static CODEC field indicates that this provider is designed to be selected and configured dynamically at server startup. A server administrator can choose this disk-based provider over other potential implementations (e.g., a database provider) through server configuration files. The provider itself acts as a factory, creating instances of the nested DiskPlayerStorage class, which performs the actual file operations.

### Lifecycle & Ownership
- **Creation:** An instance of DiskPlayerStorageProvider is not created directly via its constructor. Instead, it is instantiated by the server's configuration loading system, which uses the provided CODEC to deserialize its settings, such as the target storage path. This typically occurs once during the server's bootstrap sequence.
- **Scope:** The provider object is a long-lived service. It persists for the entire server session after being created at startup.
- **Destruction:** The object has no explicit destruction or close method. It is eligible for garbage collection when the server shuts down and its parent context is released.

## Internal State & Concurrency
- **State:** The primary state is the `path` field, which points to the directory where player files are stored. This field is set during deserialization at creation and is considered effectively immutable for the object's lifetime. The nested DiskPlayerStorage class also holds a reference to this path.
- **Thread Safety:** This component is designed for a high-concurrency environment. All I/O operations—load, save, and remove—are fully asynchronous, returning a CompletableFuture. This prevents the calling thread (e.g., the main server tick loop) from blocking on slow disk operations. The underlying I/O is managed by the BsonUtil, which presumably uses a dedicated thread pool. The class itself is stateless beyond its initial configuration, making it safe to be shared and called from multiple threads simultaneously.

## API Surface
The primary API is exposed through the nested **DiskPlayerStorage** class, which is obtained by calling **getPlayerStorage**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerStorage() | PlayerStorage | O(1) | Factory method. Returns a new DiskPlayerStorage instance for performing I/O. |
| load(uuid) | CompletableFuture | I/O Bound | Asynchronously reads a player's data file from disk and deserializes it into an EntityStore. |
| save(uuid, holder) | CompletableFuture | I/O Bound | Asynchronously serializes an EntityStore to BSON and writes it to the player's data file. |
| remove(uuid) | CompletableFuture | I/O Bound | Asynchronously deletes a player's data file from disk. |
| getPlayers() | Set | O(N) | **Blocking I/O.** Scans the storage directory and returns the UUIDs of all players with data files. Throws IOException on failure. |

## Integration Patterns

### Standard Usage
The provider is typically retrieved from a central service registry. The consumer then gets a storage instance and uses its asynchronous methods to manage player data, chaining subsequent logic to the CompletableFuture.

```java
// Assumes a context object that manages server services
PlayerStorageProvider provider = context.getService(PlayerStorageProvider.class);
PlayerStorage storage = provider.getPlayerStorage();

// Asynchronously load a player and log in
UUID playerUuid = ...;
storage.load(playerUuid).thenAccept(entityStoreHolder -> {
    // This code executes on a worker thread when the load is complete
    gameLogic.spawnPlayer(playerUuid, entityStoreHolder);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DiskPlayerStorageProvider()` or `new DiskPlayerStorage()`. The system is designed for configuration-driven instantiation to allow for different storage backends.
- **Blocking on Game Thread:** Never call `future.join()` or `future.get()` on a future returned by this class from a performance-critical thread like the main server tick loop. This will freeze the server until the disk I/O completes.
- **Calling getPlayers Frequently:** The getPlayers method is a blocking operation that scans the filesystem. It is intended for administrative tasks or one-time startup routines, not for frequent use during gameplay.

## Data Pipeline
The flow of data for serialization (saving) and deserialization (loading) is linear and straightforward.

> **Save Flow:**
> In-Memory Holder<EntityStore> -> EntityStore.REGISTRY (Serialize) -> BsonDocument -> BsonUtil (Write) -> **DiskPlayerStorage** -> Filesystem (UUID.json)

> **Load Flow:**
> Filesystem (UUID.json) -> **DiskPlayerStorage** -> BsonUtil (Read) -> BsonDocument -> EntityStore.REGISTRY (Deserialize) -> In-Memory Holder<EntityStore>

