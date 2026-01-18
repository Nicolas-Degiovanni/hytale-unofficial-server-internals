---
description: Architectural reference for DiskDataStore
---

# DiskDataStore<T>

**Package:** com.hypixel.hytale.server.core.universe.datastore
**Type:** Transient / Service

## Definition
```java
// Signature
public class DiskDataStore<T> implements DataStore<T> {
```

## Architecture & Concepts

The DiskDataStore is a concrete implementation of the DataStore persistence contract. Its primary role is to provide a durable, file-system-based storage mechanism for arbitrary server-side game objects, mapping a string identifier to a corresponding JSON file on disk.

Architecturally, this class acts as a low-level persistence adapter. It decouples higher-level game systems, such as a PlayerManager or WorldChunkManager, from the underlying details of file I/O, serialization, and data encoding. Each instance of DiskDataStore manages a single directory, which can be thought of as a "table" or "collection" for a specific data type.

The core of its operation relies on a provided **BuilderCodec**. This codec is a strategy object that defines how to translate a Java object of type T into a BsonDocument and back. This design makes the DiskDataStore highly generic and reusable for any data type that can be serialized.

All file paths are resolved relative to the server's root `Universe` path, ensuring that data is stored in a predictable and organized location within the server's file structure. The class also contains legacy migration logic to automatically upgrade older `.bson` files to the current `.json` format, indicating its role in the evolution of the server's storage engine.

### Lifecycle & Ownership
- **Creation:** A DiskDataStore is not a global singleton. It is instantiated directly by a manager or service that requires persistence for a specific data type. For example, a `GuildManager` would create its own `new DiskDataStore<Guild>("guilds", guildCodec)`.
- **Scope:** The lifetime of a DiskDataStore instance is bound to the object that creates it. It typically persists for the entire server session, servicing all persistence requests for its configured data type.
- **Destruction:** The object is eligible for garbage collection when its owner is destroyed, usually during server shutdown. It does not hold open file handles and has no explicit `close` or `shutdown` method.

## Internal State & Concurrency
- **State:** The DiskDataStore is effectively stateless concerning the data it manages; it holds no in-memory caches of objects. Its internal state consists of its configuration—the `path` and `codec`—which are final and established at construction. The true state is externalized to the file system.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be operated by a single thread or with external synchronization.
    - Concurrent write operations (save, remove, removeAll) on the same instance will lead to race conditions and data corruption.
    - While the internal encoding process uses a ThreadLocal, the file system operations themselves are not synchronized.
    - **WARNING:** All access from multiple threads must be coordinated through an external locking mechanism. Failure to do so will result in unpredictable behavior and data loss.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| load(id) | T | O(D) | Reads and deserializes a single entity from disk. Returns null if not found. Throws IOException on read failure. D represents disk I/O time. |
| save(id, value) | void | O(D) | Serializes and writes a single entity to disk, overwriting any existing file. This is a blocking operation. |
| remove(id) | void | O(D) | Deletes the data file and its corresponding backup file from disk. |
| list() | List<String> | O(N) | Scans the directory and returns a list of all entity IDs. N is the number of entities. |
| loadAll() | Map<String, T> | O(N * D) | Loads and deserializes all entities in the directory. **WARNING:** This is a high-cost operation and should be used sparingly on large data sets. |
| removeAll() | void | O(N) | Deletes all data and backup files in the directory. **WARNING:** This is a destructive and irreversible operation. |

## Integration Patterns

### Standard Usage

The DiskDataStore is intended to be encapsulated within a higher-level manager that controls the persistence logic for a specific domain.

```java
// Example within a hypothetical FactionManager
// Codec is defined elsewhere and passed in
BuilderCodec<Faction> factionCodec = Faction.CODEC;

// 1. Create a dedicated DataStore for factions
DataStore<Faction> factionStore = new DiskDataStore<>("factions", factionCodec);

// 2. Use the store to save a new faction
Faction newFaction = new Faction("Knights of Hytale");
factionStore.save(newFaction.getId(), newFaction);

// 3. Load a faction when needed
try {
    Faction loadedFaction = factionStore.load("knights-of-hytale-id");
    if (loadedFaction != null) {
        // ... logic with the loaded faction
    }
} catch (IOException e) {
    // Handle file read errors
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Writes:** Do not call `save` or `remove` from multiple threads on the same DiskDataStore instance without an external lock. This is the most common source of data corruption.
- **Frequent `loadAll` Calls:** Avoid calling `loadAll` in performance-critical code paths or game loops. The method performs a full directory scan and deserializes every file, which is extremely inefficient for frequent use. For managed collections, load all data once at startup and maintain it in memory.
- **Storing Volatile Data:** The DiskDataStore is for durable persistence. Do not use it for transient or high-frequency state changes, as the disk I/O overhead is significant.

## Data Pipeline

The flow of data for read and write operations is linear and straightforward.

> **Write Flow:**
> In-Memory Object (T) -> `BuilderCodec.encode()` -> BsonDocument -> `BsonUtil.writeDocument()` -> **DiskDataStore** -> File System (.json)

> **Read Flow:**
> File System (.json) -> **DiskDataStore** -> `RawJsonReader.readSync()` -> `BuilderCodec.decode()` -> In-Memory Object (T)

