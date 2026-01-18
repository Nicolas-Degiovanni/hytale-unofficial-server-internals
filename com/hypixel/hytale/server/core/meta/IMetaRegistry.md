---
description: Architectural reference for IMetaRegistry
---

# IMetaRegistry<K>

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface IMetaRegistry<K> {
```

## Architecture & Concepts
The IMetaRegistry interface defines a foundational contract for a type-safe metadata system. It establishes a centralized registry for defining and creating metadata objects that can be attached to a parent object of type K. This pattern is a core component of the engine's entity-component-system (ECS) or entity-attribute-value (EAV) design, providing a structured and extensible alternative to using raw HashMaps for dynamic properties.

The system revolves around three key abstractions:
1.  **IMetaRegistry:** The central authority where metadata types are defined. Implementations of this interface act as a factory for MetaKeys.
2.  **MetaKey<T>:** A typed, opaque handle representing a specific piece of metadata. It ensures compile-time type safety when accessing metadata.
3.  **IMetaStore<K>:** The concrete data container, associated with an instance of K, that holds the actual metadata values. The IMetaRegistry is used to populate and iterate over an IMetaStore.

A critical feature of this architecture is its support for data persistence. By associating a Codec with a MetaKey during registration, the system gains the ability to serialize and deserialize metadata, which is essential for saving game state, player data, or world information.

## Lifecycle & Ownership
As an interface, IMetaRegistry does not have a concrete lifecycle. However, any implementation is expected to adhere to the following lifecycle pattern.

-   **Creation:** An implementation of IMetaRegistry is instantiated once during the application's bootstrap phase (e.g., by ServerEntryPoint or ClientEntryPoint). It is a foundational service that must exist before most other systems are initialized.
-   **Scope:** The registry is a long-lived, session-scoped object. It persists for the entire lifetime of the server or client process. All metadata definitions are registered during an initialization phase and are considered immutable thereafter.
-   **Destruction:** The registry is destroyed during the final stages of application shutdown. There is typically no complex cleanup logic required, as it primarily holds definitions and factory functions.

**WARNING:** Registering new metadata types after the initial bootstrap phase is an unsupported and dangerous operation that can lead to race conditions and inconsistent state across the system.

## Internal State & Concurrency
The interface itself is stateless. Any conforming implementation, however, is inherently stateful.

-   **State:** An implementation must maintain an internal collection of all registered MetaKeys, mapping them to their corresponding factory functions, persistence keys, and codecs. This state is populated via the registerMetaObject method and is considered write-once, read-many.
-   **Thread Safety:** The interface does not enforce thread safety, but any production-grade implementation **must be thread-safe**. The registration methods are typically called from a single thread during startup. However, methods like newMetaObject and getMetaKeyForCodecKey may be called from multiple threads (e.g., network threads deserializing data, game logic threads creating new entities). Implementations should use concurrent collections or appropriate locking mechanisms to protect their internal state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerMetaObject(...) | MetaKey<T> | O(1) | Registers a new metadata type. Returns a typed key for future access. This is the primary configuration method. |
| newMetaObject(key, parent) | T | O(1) | Instantiates a new metadata object for a given parent object using the registered factory function. |
| forEachMetaEntry(store, consumer) | void | O(N) | Iterates over all metadata entries within a given IMetaStore. N is the number of entries in the store. |
| getMetaKeyForCodecKey(key) | PersistentMetaKey<?> | O(1) | Retrieves a registered MetaKey using its string-based persistence key. Essential for deserialization. |

## Integration Patterns

### Standard Usage
The primary pattern is to register all required metadata types during a system's initialization phase and store the returned MetaKey handles in static fields for later use.

```java
// In a system's initialization block (e.g., PlayerSystem)
public static final MetaKey<PlayerStats> PLAYER_STATS;

static {
    IMetaRegistry<Player> registry = ServiceRegistry.get(IMetaRegistry.class);
    PLAYER_STATS = registry.registerMetaObject(
        player -> new PlayerStats(player.getUUID()), // Factory function
        "player_stats",                             // Persistence key
        PlayerStats.CODEC                           // Serializer/Deserializer
    );
}

// Later, when accessing metadata on a player's IMetaStore
PlayerStats stats = player.getMetaStore().get(PLAYER_STATS);
stats.incrementKills();
```

### Anti-Patterns (Do NOT do this)
-   **Dynamic Registration:** Do not call registerMetaObject outside of a controlled, single-threaded initialization phase. Registering keys during runtime can cause severe concurrency issues and state desynchronization.
-   **Ignoring the Key:** Do not discard the MetaKey returned by registration. Attempting to re-register the same metadata type will lead to unpredictable behavior or errors.
-   **Stateless Factories:** Avoid using factory functions (the `Function<K, T>`) that rely on mutable external state. They should be pure functions or simple constructors to ensure predictable object creation.

## Data Pipeline
The IMetaRegistry is central to the metadata serialization and deserialization pipeline.

> **Serialization Flow:**
> Game Save Trigger -> Entity Loop -> Get IMetaStore -> **IMetaRegistry.forEachMetaEntry** -> For each entry, get MetaKey -> Get Codec from MetaKey -> Codec.encode(value) -> Write to disk

> **Deserialization Flow:**
> Read from disk -> Get persistence key (e.g., "player_stats") -> **IMetaRegistry.getMetaKeyForCodecKey(key)** -> Get Codec from MetaKey -> Codec.decode(data) -> Populate new IMetaStore with key and decoded value

