---
description: Architectural reference for IMetaStore
---

# IMetaStore

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IMetaStore<K> {
```

## Architecture & Concepts

The IMetaStore interface defines a contract for attaching arbitrary, type-safe data to a host object. It is a core component of the server's data attachment system, enabling dynamic extension of game objects like entities, players, or worlds without modifying their base classes. This pattern is commonly known as the "Metadata" or "Attachment" pattern.

Architecturally, IMetaStore acts as a clean, public-facing facade. All of its operations are delegated to a concrete IMetaStoreImpl instance, which contains the actual storage and logic. This is an application of the **Bridge Pattern**, decoupling the abstraction (the contract for storing metadata) from its implementation (the actual data structure, likely a hash map). This design allows the underlying storage mechanism to be changed or optimized without affecting any of the systems that interact with the IMetaStore interface.

The generic parameter K typically represents the type of the object that *owns* the metadata, providing a degree of self-reference and context. Type safety for the stored data is achieved through the use of MetaKey objects, where each key is intrinsically linked to the type of value it represents.

## Lifecycle & Ownership

As an interface, IMetaStore does not have a lifecycle of its own. Instead, its lifecycle is bound to the object that implements it.

-   **Creation:** The concrete IMetaStoreImpl is instantiated and managed by the host object (e.g., an Entity or Player) during its own construction. The host object is the sole owner of its metadata store.
-   **Scope:** The metadata and its container persist for the exact lifetime of the host object. If a Player object exists, its IMetaStore exists.
-   **Destruction:** The metadata is eligible for garbage collection when its owning host object is destroyed or unloaded. There is no separate cleanup or teardown process for the metadata store itself; it lives and dies with its owner.

## Internal State & Concurrency

-   **State:** The interface itself is stateless. All state is managed by the backing IMetaStoreImpl instance. This implementation is inherently **mutable**, as its primary function is to serve as a dynamic, key-value cache for runtime data.
-   **Thread Safety:** **Not thread-safe.** The IMetaStore contract provides no guarantees of concurrent access safety. All interactions with an object's metadata store **must** be performed on the primary thread responsible for that object's updates (e.g., the main server game loop). Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. Any required synchronization must be handled externally by the calling system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getMetaStore() | IMetaStoreImpl | O(1) | Returns the backing implementation. **Warning:** For internal engine use only. |
| getMetaObject(key) | T | O(1) | Retrieves an object by its type-safe key. Throws an exception if not found. |
| getIfPresentMetaObject(key) | T | O(1) | Retrieves an object if it exists, otherwise returns null. |
| putMetaObject(key, obj) | T | O(1) | Associates a value with a key. Returns the previous value, if any. |
| removeMetaObject(key) | T | O(1) | Removes an object by its key. |
| markMetaStoreDirty() | void | O(1) | Flags the store as modified, signaling to persistence or replication systems. |
| consumeMetaStoreDirty() | boolean | O(1) | Checks if the store is dirty and atomically resets the flag to false. |

## Integration Patterns

### Standard Usage

The primary use case is to attach and retrieve system-specific data from a generic game object. A system defines its own MetaKey and uses it to access its data on any object that implements IMetaStore.

```java
// Assume 'player' is an object that implements IMetaStore
// QuestSystem defines a static key for its data
public static final MetaKey<QuestProgress> QUEST_PROGRESS = ...;

// Storing data
QuestProgress progress = new QuestProgress();
player.putMetaObject(QUEST_PROGRESS, progress);
player.markMetaStoreDirty(); // Signal that player data needs saving

// Retrieving data later
QuestProgress progress = player.getMetaObject(QUEST_PROGRESS);
if (progress != null) {
    progress.advanceStage();
    player.markMetaStoreDirty();
}
```

### Anti-Patterns (Do NOT do this)

-   **Implementation-Specific Calls:** Do not retrieve the backing store via getMetaStore and interact with it directly. This violates the design principle and couples your system to a specific implementation, making future engine upgrades brittle.
-   **Cross-Thread Modification:** Never call putMetaObject or removeMetaObject from an asynchronous task or different thread without external locking. This will corrupt the metadata map.
-   **Forgetting to Mark Dirty:** Failing to call markMetaStoreDirty after a modification means the changes may never be persisted to the database or replicated to clients, leading to data loss.

## Data Pipeline

IMetaStore is a critical component in the server's data persistence and replication pipeline. The dirty-flagging mechanism is central to this process.

> **Flow: Data Modification and Persistence**
>
> 1.  An external system (e.g., Quest System) modifies an object's state via `putMetaObject`.
> 2.  The system immediately calls `markMetaStoreDirty()` on the same object.
> 3.  A separate, global system (e.g., WorldSaveManager, PlayerPersistenceService) periodically iterates over all relevant objects.
> 4.  For each object, the service calls `consumeMetaStoreDirty()`.
> 5.  If this returns true, the service knows the metadata has changed and triggers serialization of the **entire** metadata store for that object, writing it to disk or a database.

