---
description: Architectural reference for DynamicMetaStore
---

# DynamicMetaStore

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Transient

## Definition
```java
// Signature
public class DynamicMetaStore<K> extends AbstractMetaStore<K> {
```

## Architecture & Concepts

The DynamicMetaStore is a concrete implementation of AbstractMetaStore, designed to provide a flexible, in-memory container for arbitrary metadata attached to a parent object. It serves as the standard, general-purpose metadata holder for server-side objects like entities, players, or world chunks.

Its core architectural characteristic is the use of a fastutil Int2ObjectOpenHashMap. This provides a highly optimized, heap-allocated key-value store. The "Dynamic" in its name refers to this ability to grow and shrink its storage as metadata is added or removed, contrasting with potentially more rigid, array-backed implementations.

This class operates as a satellite component. It does not exist on its own but is tightly coupled to a parent object of type K, holding state *on behalf of* that parent. All key-to-integer-ID translation is delegated to an IMetaRegistry instance provided during construction, making the registry a critical dependency for the store's operation.

A key behavior is its implementation of a **lazy-load** or **cache-on-read** pattern. The `getMetaObject` method will first check for a value in its internal map. On a cache miss, it invokes `decodeOrNewMetaObject` (inherited from its parent) to either deserialize the data or create a default instance, which is then stored for subsequent requests.

### Lifecycle & Ownership
- **Creation:** A DynamicMetaStore is not intended to be instantiated directly by feature developers. It is created by a managing system responsible for the parent object's lifecycle. For example, an EntityManager will construct a new DynamicMetaStore when it creates a new Entity.
- **Scope:** The lifecycle of a DynamicMetaStore is strictly bound to its parent object. It is created when the parent is created and persists for the exact duration of the parent's existence.
- **Destruction:** The object is cleaned up by the Java garbage collector when its parent object is destroyed and all external references to the store are released. It does not require an explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The DynamicMetaStore is a highly mutable, stateful component. Its primary state is the internal `map` which caches all metadata objects. This state is modified on nearly every public method call, including `put`, `remove`, and the initial `get` for a given key.
- **Thread Safety:** **This class is not thread-safe.** The internal map is a standard, non-concurrent HashMap. Concurrent read and write operations from multiple threads will lead to data corruption, race conditions, or a ConcurrentModificationException.

    **WARNING:** All interactions with a DynamicMetaStore instance **must** be performed on the owning thread, which is typically the main server game-tick thread. Do not share instances across threads without implementing external locking.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMetaObject(MetaKey<T> key) | T | Amortized O(1) | Retrieves the metadata object for the given key. Lazily decodes or creates the object on first access. |
| putMetaObject(MetaKey<T> key, T obj) | T | O(1) | Associates the object with the key and marks the store as dirty for persistence or networking. |
| removeMetaObject(MetaKey<T> key) | T | O(1) | Removes the metadata for the given key and marks the store as dirty. |
| hasMetaObject(MetaKey<?> key) | boolean | O(1) | Checks for the presence of a metadata object for the key without triggering lazy-loading. |
| clone(K parent) | DynamicMetaStore<K> | O(N) | Creates a new store for a new parent, performing a shallow copy of all contained metadata entries. |
| copyFrom(DynamicMetaStore<K> other) | void | O(N) | Copies all metadata entries from another store into this one. Throws if the registries do not match. |

## Integration Patterns

### Standard Usage
The DynamicMetaStore should always be retrieved from its parent object. Direct interaction involves using predefined MetaKey constants to get or set typed data.

```java
// Assume 'player' is an object that provides access to its MetaStore
IMetaStore<Player> meta = player.getMetaStore();

// Set a value using a predefined key
meta.putMetaObject(PlayerMetaKeys.LAST_LOGIN_TIMESTAMP, System.currentTimeMillis());

// Retrieve the value. The store handles the type casting.
long lastLogin = meta.getMetaObject(PlayerMetaKeys.LAST_LOGIN_TIMESTAMP);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DynamicMetaStore()`. The framework is responsible for creating and attaching stores to their parent objects. Doing so will result in a disconnected state that is not persisted or synchronized.
- **Cross-Thread Modification:** Do not access and modify a store from an asynchronous task or a different thread without explicit, external synchronization. This will corrupt the store's internal state.
- **Registry Mismatch:** Attempting to use the `copyFrom` method with a store that was initialized with a different IMetaRegistry instance is a fatal error and will throw an IllegalArgumentException.

## Data Pipeline

The DynamicMetaStore is a critical component in the server's data persistence and network synchronization pipelines. Its role is to act as a mutable buffer for an object's state, signaling when that state has changed.

> **Write Flow:**
> Game Logic -> `putMetaObject()` / `removeMetaObject()` -> **DynamicMetaStore** (updates internal map) -> `markMetaStoreDirty()` -> Persistence/Network System (detects dirty flag) -> Serialize Store -> Write to Database / Send Network Packet

> **Read Flow (Lazy Load):**
> Game Logic -> `getMetaObject()` -> **DynamicMetaStore** (cache miss) -> `decodeOrNewMetaObject()` -> Read from Database / Use Factory Default -> **DynamicMetaStore** (caches new object) -> Return to Game Logic

