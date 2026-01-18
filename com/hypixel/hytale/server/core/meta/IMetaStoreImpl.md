---
description: Architectural reference for IMetaStoreImpl
---

# IMetaStoreImpl

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Contract Interface

## Definition
```java
// Signature
public interface IMetaStoreImpl<K> extends IMetaStore<K> {
```

## Architecture & Concepts
The IMetaStoreImpl interface defines the low-level contract for components responsible for the serialization and deserialization of game-state metadata. It acts as the critical translation layer between the server's in-memory object representation and its persistent BSON format, which is used for database storage and potentially network replication.

This interface is not intended for direct use by high-level game logic. Instead, it provides the core implementation hooks for concrete metadata stores, such as PlayerMetaStore or WorldEntityMetaStore. Its primary responsibility is to manage the bidirectional flow of data, converting complex Java objects into a BSON document upon saving (encoding) and reconstructing the object state from a BSON document upon loading (decoding).

The inclusion of the ExtraInfo parameter in the encode/decode methods indicates that serialization is context-sensitive. This allows the system to alter the serialization output based on factors like the target client version, network conditions, or storage destination (e.g., full save vs. incremental update).

## Lifecycle & Ownership
As an interface, IMetaStoreImpl does not have a lifecycle of its own. The lifecycle described here pertains to the concrete classes that **implement** this contract.

-   **Creation:** An implementing object is typically instantiated by a parent factory or manager when the associated game object (e.g., a Player, an NPC, a WorldChunk) is created or loaded into memory. For example, a PlayerMetaStore is created by the PlayerLifecycleService when a player joins the server.
-   **Scope:** The lifetime of an implementing object is strictly tied to the game object it represents. It persists as long as the parent object is active in the server's memory.
-   **Destruction:** The object is marked for garbage collection when its parent game object is unloaded or destroyed. Explicit cleanup is generally not required, but implementations may be pooled or reset by their owning system to reduce allocation overhead.

**WARNING:** Implementations of this interface often hold the canonical state for a game object. Premature destruction or loss of reference will result in immediate data loss.

## Internal State & Concurrency
The interface itself is stateless. However, any non-trivial implementation will be stateful, holding the metadata it is designed to manage.

-   **State:** Implementations are expected to be highly mutable. The decode method populates the internal state from an external data source, and subsequent game logic operations will modify this state before it is written out via the encode method.
-   **Thread Safety:** This contract makes no guarantees about thread safety. It is the sole responsibility of the implementing class to ensure safe concurrent access. Given that metadata can be accessed by game logic threads, network threads, and persistence threads, implementations **must** be designed to be thread-safe, typically through the use of synchronized blocks, concurrent collections, or atomic references. Failure to do so will lead to state corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRegistry() | IMetaRegistry<K> | O(1) | Returns the registry associated with this store, used for resolving metadata types. |
| decode(doc, info) | void | O(N) | Populates the internal state of the object from a BSON document. N is the number of keys in the document. |
| encode(info) | BsonDocument | O(M) | Serializes the current internal state into a new BSON document. M is the number of managed metadata fields. |
| forEachUnknownEntry(consumer) | void | O(U) | Iterates over key-value pairs from the last decode operation that were not recognized. U is the number of unknown keys. Critical for data migration and forward compatibility. |

## Integration Patterns

### Standard Usage
Implementations of IMetaStoreImpl are managed by higher-level services. Game logic interacts with the metadata through the simpler IMetaStore interface, while the persistence layer uses the IMetaStoreImpl interface to trigger serialization.

```java
// Example: A persistence service saving a player's metadata
public void savePlayer(Player player) {
    // Retrieve the concrete implementation
    IMetaStoreImpl<PlayerMetaKey> metaStore = player.getMetaStoreImpl();

    // Create context for the serialization operation
    ExtraInfo saveContext = createSaveContext();

    // Encode the current state into BSON
    BsonDocument playerData = metaStore.encode(saveContext);

    // Write the document to the database
    database.save("players", player.getUUID(), playerData);
}
```

### Anti-Patterns (Do NOT do this)
-   **State Bypass:** Do not modify the internal state of an implementation and then fail to call encode. The persistence layer relies on the BSON document generated by encode as the single source of truth for storage.
-   **Ignoring Unknown Entries:** Failing to handle or log entries passed to forEachUnknownEntry can lead to silent data loss when the server is updated. Unrecognized data from a newer client or server version will be discarded on the next save.
-   **Incorrect Context:** Passing a default or incorrect ExtraInfo object to encode or decode can result in data being serialized improperly for the target destination, leading to compatibility issues or data corruption.

## Data Pipeline
IMetaStoreImpl is a key component in the data persistence and loading pipeline.

> **Load Flow:**
> Database Read -> BSON Document -> **decode(doc)** -> In-Memory Java Object State -> Game Logic

> **Save Flow:**
> Game Logic (mutates state) -> In-Memory Java Object State -> **encode()** -> BSON Document -> Database Write

