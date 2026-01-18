---
description: Architectural reference for ArrayMetaStore
---

# ArrayMetaStore

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Transient State Container

## Definition
```java
// Signature
public class ArrayMetaStore<K> extends AbstractMetaStore<K> {
```

## Architecture & Concepts
The ArrayMetaStore is a high-performance, specialized implementation of the AbstractMetaStore. Its primary architectural function is to provide extremely fast, O(1) access to metadata associated with a parent object, such as an entity or a world chunk.

The core design principle is the direct mapping of a MetaKey's integer ID to an index in an internal `Object[]` array. This strategy completely bypasses the computational overhead of hash code calculation and bucket collision resolution found in traditional HashMap-based stores. This makes it ideal for performance-critical systems where metadata is frequently accessed, such as during the main game loop.

However, this performance comes with a significant trade-off: memory usage. The internal array is dynamically resized to accommodate the highest MetaKey ID requested. If the key IDs are sparse (e.g., 1, 5, 10000), the array will grow to the largest ID, leaving the intermediate indices empty. This can lead to substantial memory waste if not managed carefully by the key registry system.

The class uses a static sentinel object, NO_ENTRY, to distinguish between a slot that has never been written to and a slot that explicitly contains a null value.

## Lifecycle & Ownership
- **Creation:** An ArrayMetaStore is not intended for direct instantiation by developers. It is created by a higher-level metadata management system (e.g., an EntityManager or WorldManager) when metadata capabilities are required for a parent object. The parent object and the central IMetaRegistry are injected via the constructor.
- **Scope:** The lifecycle of an ArrayMetaStore is strictly tied to its parent object. It exists for as long as the parent object is active in the game state.
- **Destruction:** The object is eligible for garbage collection as soon as its parent object is dereferenced and no other systems hold a direct reference to the store. It has no explicit cleanup or `close` method.

## Internal State & Concurrency
- **State:** The ArrayMetaStore is a highly mutable, stateful container. Its primary state is the internal `Object[]` array, which grows on-demand but never shrinks. The state is marked as dirty via `markMetaStoreDirty` on writes and removals, signaling to the persistence layer that the parent object's data needs to be saved.
- **Thread Safety:** **This class is not thread-safe.** All internal operations, particularly `resizeArray`, are non-atomic. Concurrent access from multiple threads will result in race conditions, `ArrayIndexOutOfBoundsException`, data corruption, or lost updates. All interactions with an ArrayMetaStore instance must be synchronized externally or, more commonly, confined to a single, designated thread such as the main server tick thread.

## API Surface
The public API is designed for retrieving, storing, and checking for the existence of metadata objects.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMetaObject(MetaKey) | T | O(1) avg, O(N) worst | Retrieves the metadata object. Triggers lazy-deserialization and array resizing if the key is not present. |
| putMetaObject(MetaKey, T) | T | O(1) avg, O(N) worst | Associates a value with a key. Marks the store as dirty. May trigger an expensive array resize. |
| removeMetaObject(MetaKey) | T | O(1) | Removes a value by replacing it with the NO_ENTRY sentinel. Marks the store as dirty. Does not shrink the array. |
| hasMetaObject(MetaKey) | boolean | O(1) | Checks for the presence of a key without triggering deserialization. |
| getIfPresentMetaObject(MetaKey) | T | O(1) | Retrieves the metadata object only if it is already present in the array. Returns null otherwise. |

## Integration Patterns

### Standard Usage
The ArrayMetaStore should never be instantiated directly. It is retrieved from its parent object, which is responsible for its creation and lifecycle. The primary interaction is through `getMetaObject` and `putMetaObject`.

```java
// Assume 'entity' is a game object that uses an ArrayMetaStore internally
MetaStore store = entity.getMetaStore();

// Writing data
store.putMetaObject(MetaKeys.CURRENT_HEALTH, 95);

// Reading data
Integer health = store.getMetaObject(MetaKeys.CURRENT_HEALTH);
if (health != null) {
    // ... process health value
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ArrayMetaStore()`. The store will not be correctly registered with the persistence system or its parent, leading to data loss and runtime exceptions.
- **Multi-threaded Access:** Do not share an ArrayMetaStore instance across threads without external locking. This will corrupt the internal array and cause unpredictable behavior.
- **Storing Volatile State:** While fast, this store is tied to a persistence system via `markMetaStoreDirty`. Do not use it for purely transient, high-frequency state that does not need to be saved, as this will generate unnecessary serialization overhead.

## Data Pipeline
The ArrayMetaStore acts as a caching and storage layer in the broader metadata system. The data flow for a cache miss (lazy-loading) is as follows.

> Flow:
> Game Logic requests data via `getMetaObject` -> **ArrayMetaStore** (cache miss, index out of bounds or contains NO_ENTRY) -> `AbstractMetaStore.decodeOrNewMetaObject` -> `IMetaRegistry` (handles deserialization from a persistent source) -> **ArrayMetaStore** (receives new object, populates it into the resized array) -> Game Logic receives data

