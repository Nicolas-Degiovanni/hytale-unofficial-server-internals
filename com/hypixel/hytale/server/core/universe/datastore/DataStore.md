---
description: Architectural reference for the DataStore interface, the core abstraction for server-side persistence.
---

# DataStore<T>

**Package:** com.hypixel.hytale.server.core.universe.datastore
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface DataStore<T> {
```

## Architecture & Concepts
The DataStore interface is a foundational abstraction within the server's persistence layer. It defines a generic contract for a key-value storage system, where keys are represented by a String identifier and values are generic, serializable objects of type T.

Its primary architectural purpose is to decouple high-level game systems (such as player profile management, world state persistence, or guild data) from the low-level implementation details of data storage. This allows the underlying storage mechanism—be it a local file system, a relational database, or a distributed cache—to be swapped without altering the game logic that consumes the data.

A critical component of this contract is the association with a `BuilderCodec<T>`. This mandates that any object type T managed by a DataStore must have a corresponding codec capable of serializing it to a byte stream and deserializing it back into an object. This design enforces a clean separation between the data's in-memory representation and its on-disk format.

Implementations of DataStore are expected to be specialized for specific data types, for example, a `PlayerDataStore` implementing `DataStore<PlayerData>`.

## Lifecycle & Ownership
As an interface, DataStore itself has no lifecycle. The following describes the expected lifecycle of its concrete implementations.

- **Creation:** Implementations are typically instantiated by a central service registry or a dedicated `DataStoreManager` during server bootstrap. Each store is configured with its specific type T and the corresponding codec.
- **Scope:** A DataStore instance is a long-lived object, persisting for the entire server session. It acts as a singleton for a specific type of data (e.g., one instance for all player data, one for all guild data).
- **Destruction:** The instance is eligible for garbage collection only upon server shutdown when the managing service releases all references. There is no explicit `close` or `shutdown` method in the interface, implying that resource management is the responsibility of the implementation or the factory that creates it.

## Internal State & Concurrency
- **State:** The interface is stateless. However, all concrete implementations are inherently stateful, as they manage and provide access to a persistent collection of data. Implementations may employ in-memory caching strategies to improve read performance, but this is not enforced by the contract.
- **Thread Safety:** **WARNING:** The DataStore interface does **not** guarantee thread safety. It is the responsibility of the concrete implementation to ensure that concurrent read/write operations are handled safely, typically through synchronization mechanisms like locks or concurrent data structures. Consumers of this interface should assume that concurrent access to the same key is unsafe unless the specific implementation guarantees otherwise.

## API Surface
The public contract focuses on atomic and bulk data manipulation operations. All methods that interact with the underlying storage are declared to throw IOException, forcing callers to handle potential persistence failures.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCodec() | BuilderCodec<T> | O(1) | Returns the codec used for serializing and deserializing objects of type T. |
| load(String id) | T | O(1) | Retrieves and deserializes a single object by its unique identifier. Returns null if not found. |
| save(String id, T object) | void | O(1) | Serializes and saves a single object, associating it with the given identifier. |
| remove(String id) | void | O(1) | Deletes the object associated with the given identifier from the persistent store. |
| list() | List<String> | O(N) | Returns a list of all known identifiers in the store. **Warning:** This can be a very expensive operation. |
| loadAll() | Map<String, T> | O(N) | Loads all objects from the store into a map. Uses `list()` and `load()` internally. |
| saveAll(Map<String, T> map) | void | O(N) | Saves all entries from the provided map to the store. |
| removeAll() | void | O(N) | Deletes all objects from the store. **Warning:** This is a destructive, high-cost operation. |

## Integration Patterns

### Standard Usage
A service responsible for a specific domain model (e.g., PlayerData) will obtain a pre-configured DataStore instance from a central registry and use it for all persistence operations. Error handling for IOException is mandatory.

```java
// A service obtains its specific DataStore from a manager or context
DataStore<PlayerData> playerStore = dataStoreManager.getStore(PlayerData.class);

try {
    // Load a player's data on login
    PlayerData data = playerStore.load("player-uuid-123");
    if (data == null) {
        data = new PlayerData(); // Create new profile
    }

    // Modify and save the data
    data.setLastLogin(System.currentTimeMillis());
    playerStore.save("player-uuid-123", data);

} catch (IOException e) {
    // Critical error: handle persistence failure (e.g., log, kick player)
    log.error("Failed to access player data store", e);
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring IOExceptions:** Swallowing or failing to properly handle an IOException can lead to silent data loss or corruption. All calls to `load`, `save`, and `remove` must be wrapped in appropriate error handling logic.
- **Frequent `list()` or `loadAll()` Calls:** These methods are not designed for high-frequency use in the main game loop. They iterate over the entire dataset and can cause significant I/O load and performance degradation. Use them for administrative tasks, server startup, or batch processing only.
- **Assuming Transactionality:** The interface provides no transactional guarantees. A call to `saveAll` is not atomic; a failure midway through the operation may leave the store in a partially updated state. Complex, multi-key operations that must be atomic require a higher-level transaction management system.

## Data Pipeline
The DataStore interface sits at the boundary between the in-memory game state and the serialized, persistent state.

**Write Operation (`save`)**
> Flow:
> In-Memory Game Object (T) -> **DataStore.save()** -> BuilderCodec.encode() -> Serialized Byte Array -> Underlying Storage (e.g., File System Write, Database Query)

**Read Operation (`load`)**
> Flow:
> Underlying Storage (e.g., File System Read, Database Query) -> Serialized Byte Array -> **DataStore.load()** -> BuilderCodec.decode() -> In-Memory Game Object (T)

