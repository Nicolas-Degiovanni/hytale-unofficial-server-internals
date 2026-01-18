---
description: Architectural reference for AbstractMetaStore
---

# AbstractMetaStore

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Component / Transient

## Definition
```java
// Signature
public abstract class AbstractMetaStore<K> implements IMetaStoreImpl<K> {
```

## Architecture & Concepts

The AbstractMetaStore is a foundational base class for implementing component-based metadata storage. It is not a standalone service but rather a stateful component whose lifecycle is tightly coupled to a parent object of type K, such as an Entity, Player, or World.

Its primary architectural role is to provide a robust, schema-agnostic container for arbitrary key-value data. This is achieved through a sophisticated integration with an IMetaRegistry, which defines the "schema" of known metadata keys (MetaKey).

Key architectural features include:

*   **Schema-on-Read and Forward Compatibility:** The class maintains a separate BsonDocument for `unknownValues`. When data is deserialized via the `decode` method, any key not registered in the IMetaRegistry is preserved in this document. This is a critical feature for forward compatibility, preventing data loss when a server loads data saved by a newer version of the game with additional metadata fields. These unknown fields are preserved and re-encoded upon saving.

*   **Lazy Deserialization:** Data for known keys that was initially treated as "unknown" is only fully decoded when it is first accessed. The `tryDecodeUnknownKey` method acts as a just-in-time deserialization gateway, moving data from the `unknownValues` BsonDocument into the primary typed storage on demand. This improves loading performance by deferring the cost of full object hydration.

*   **Dirty-Tracking and Caching:** The class implements a dirty-tracking mechanism (`dirty` flag) and an optional serialization cache (`cachedEncoded`). When metadata is modified, the store must be marked as dirty via `markMetaStoreDirty`. This invalidates the cache, ensuring that the next call to `encode` re-serializes the current state. This pattern significantly reduces performance overhead in systems that frequently check for data to persist, as it avoids redundant serialization work.

## Lifecycle & Ownership

*   **Creation:** An AbstractMetaStore is never instantiated directly. A concrete implementation is created by a higher-level manager or factory when its parent object (of type K) is created. It is initialized with a reference to its parent and a corresponding IMetaRegistry.

*   **Scope:** The instance's lifetime is strictly bound to its parent object. It exists solely to hold metadata for that parent and is discarded when the parent is destroyed.

*   **Destruction:** The object is reclaimed by the garbage collector when its parent is de-referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency

*   **State:** The AbstractMetaStore is highly stateful and mutable. It manages the primary metadata collection (in a concrete subclass), a BsonDocument of unrecognized data, a dirty flag, and an optional cached BsonDocument representing the serialized state.

*   **Thread Safety:** **This class is not thread-safe.** Its internal state, particularly the `dirty` flag, `cachedEncoded` document, and `unknownValues` map, are accessed and mutated without any synchronization. Concurrent read/write operations will lead to race conditions, cache corruption, and unpredictable behavior.

    **WARNING:** All interactions with an AbstractMetaStore instance must be synchronized externally or confined to a single thread, such as the main server tick loop.

## API Surface

The primary public contract is defined by the IMetaStoreImpl interface. The most critical methods for system integration are:

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| markMetaStoreDirty() | void | O(1) | Marks the store as modified. This is a mandatory call after any state change to ensure data is persisted correctly. |
| consumeMetaStoreDirty() | boolean | O(1) | Returns true if the store has been modified since the last check, and atomically resets the dirty flag to false. |
| encode(extraInfo) | BsonDocument | O(N) | Serializes the entire metadata state into a BsonDocument. If caching is enabled and the store is not dirty, this is an O(1) operation. |
| decode(document, extraInfo) | void | O(M) | Populates the store by deserializing a BsonDocument. Unrecognized keys are preserved. |
| forEachUnknownEntry(consumer) | void | O(U) | Provides a mechanism to inspect unrecognized data that has been loaded but not mapped to a known MetaKey. |

## Integration Patterns

### Standard Usage

A typical interaction involves a system retrieving the meta store from a parent object, modifying a value through a concrete implementation's method, and a separate persistence system later encoding the store if it has changed.

```java
// A game system modifies a player's metadata
Player player = getPlayer();
PlayerMetaStore metaStore = player.getMetaStore();

// This operation MUST call markMetaStoreDirty() internally
metaStore.setHealth(new Health(100));

// ... later, in a persistence or networking thread ...

// The persistence system checks if there's work to do
if (metaStore.consumeMetaStoreDirty()) {
    // The state has changed, so we serialize it for saving
    BsonDocument data = metaStore.encode(null);
    database.savePlayerData(player.getId(), data);
}
```

### Anti-Patterns (Do NOT do this)

*   **Multi-threaded Modification:** Accessing the same AbstractMetaStore instance from multiple threads without external locking will corrupt its state. Do not read metadata from a network thread while the main game thread is writing to it.

*   **Forgetting to Mark Dirty:** If a concrete subclass modifies its internal data structures but fails to call `markMetaStoreDirty()`, the `encode` method may return a stale, cached BsonDocument, leading to data loss.

*   **Relying on `encode` for Game Logic:** The `encode` method is for serialization and may return a cached value. Do not use it to get the "live" state of the object for game logic; access data through the typed getters provided by the implementation.

## Data Pipeline

The AbstractMetaStore acts as a critical translation layer between the in-memory game state and its serialized BSON representation.

### Deserialization (Loading)
> Flow:
> Database or Network -> BsonDocument -> `decode()` -> **AbstractMetaStore** -> Known data is deserialized; Unknown data is stored in `unknownValues` -> Game object is hydrated

### Serialization (Saving)
> Flow:
> Game Logic modifies data -> `markMetaStoreDirty()` is called -> Persistence System calls `encode()` -> **AbstractMetaStore** combines typed data and `unknownValues` -> BsonDocument -> Database or Network

