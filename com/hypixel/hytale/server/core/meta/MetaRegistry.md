---
description: Architectural reference for MetaRegistry
---

# MetaRegistry

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Registry Service

## Definition
```java
// Signature
public class MetaRegistry<K> implements IMetaRegistry<K> {
```

## Architecture & Concepts
The MetaRegistry is a foundational service for creating and managing metadata types within the server. It acts as a centralized factory and type registry, decoupling the definition of a metadata object from its instantiation.

The core design revolves around registering a factory function (a `java.util.function.Function`) that knows how to create a specific metadata object. In return, the registry provides a type-safe `MetaKey`. This key, not a raw string or integer, is then used throughout the game engine to request new instances of that metadata.

The generic parameter `K` represents the context or owner for which the metadata is being created. For example, a `MetaRegistry<Entity>` would be responsible for creating metadata objects that belong to entities.

This system supports two distinct kinds of metadata:

*   **Transient Metadata:** Identified by a simple integer-based `MetaKey`. These objects exist only in memory at runtime and are not intended for serialization.
*   **Persistent Metadata:** Identified by a `PersistentMetaKey`, which contains both an integer ID and a unique string name. These registrations require a `Codec` and are designed to be serialized to disk or transmitted over the network. The string name acts as a stable identifier for deserialization.

## Lifecycle & Ownership
- **Creation:** A MetaRegistry is not a global singleton. It is instantiated by a higher-level manager that governs a specific domain. For example, an `EntityManager` would create and own a `MetaRegistry<Entity>`, while a `WorldManager` might own a `MetaRegistry<World>`.
- **Scope:** The lifetime of a MetaRegistry is bound to its owning context. It persists as long as its domain is active (e.g., for the entire duration of a server session).
- **Destruction:** The object is eligible for garbage collection when its owning manager is destroyed. It holds no native resources and does not require an explicit destruction method.

## Internal State & Concurrency
- **State:** The MetaRegistry is highly stateful and mutable, but primarily during the application's initial bootstrap phase. It maintains an internal list of all registered metadata suppliers and a map for fast lookup of persistent, named metadata. The intended operational model is **write-once, read-many**; all registrations should occur at startup, after which the internal collections are treated as immutable.

- **Thread Safety:** This class is thread-safe, enforced by a `ReentrantReadWriteLock`.
    - **Registration (Write Operations):** The `registerMetaObject` method acquires an exclusive **write lock**. This serializes all registration calls, preventing data corruption.
    - **Instantiation (Read Operations):** The `newMetaObject` method acquires a shared **read lock**. This allows numerous threads to create metadata instances concurrently without contention, which is critical for server performance.

    **WARNING:** Frequent registration after the initial bootstrap phase is a severe performance anti-pattern. The write lock will block all threads attempting to create *any* metadata, potentially causing server-wide stalls.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerMetaObject(function, persistent, keyName, codec) | MetaKey | O(1) | Registers a metadata factory. Acquires a write lock. Throws IllegalStateException if a persistent keyName is duplicated. |
| newMetaObject(key, parent) | T | O(1) | Creates a new metadata instance using a registered factory. Acquires a read lock. |
| getMetaKeyForCodecKey(codecKey) | PersistentMetaKey | O(1) | Retrieves a persistent key by its stable string name. Used primarily during deserialization. |
| forEachMetaEntry(store, consumer) | void | O(N) | Iterates over entries in an IMetaStore, applying a consumer function. |

## Integration Patterns

### Standard Usage
The correct pattern is to register all metadata types during server initialization and store the returned `MetaKey` objects in static fields for later use.

```java
// During server bootstrap
// 1. Create the registry for a specific context (e.g., Entity)
MetaRegistry<Entity> entityMetaRegistry = new MetaRegistry<>();

// 2. Register a metadata type and store its key
public static final MetaKey<HealthComponent> HEALTH = entityMetaRegistry.registerMetaObject(
    (entity) -> new HealthComponent(entity, 100),
    true, // Persistent
    "hytale:health",
    HEALTH_CODEC
);

// In game logic, much later
// 3. Use the static key to create an instance for a specific entity
Entity myEntity = ...;
HealthComponent health = entityMetaRegistry.newMetaObject(HEALTH, myEntity);
myEntity.setMeta(health);
```

### Anti-Patterns (Do NOT do this)
- **Late Registration:** Do not call `registerMetaObject` after the server has finished loading. This can introduce unpredictable stalls due to write lock contention. All registrations must occur within a controlled, single-threaded bootstrap phase.
- **String-Based Lookups:** Avoid using `getMetaKeyForCodecKey` in performance-critical game logic. It is designed for deserialization, not as a primary access pattern. Always use the statically-held `MetaKey` object.
- **Ignoring Persistence Rules:** Do not register a type as persistent (`persistent = true`) without providing a valid `Codec`. The registry will throw an `IllegalStateException`.

## Data Pipeline
The MetaRegistry serves two primary data flows: defining a metadata type and instantiating it. The deserialization flow is a specialized variant of instantiation.

> **Registration Flow (Bootstrap):**
> System Initializer -> `registerMetaObject(factory, "key", codec)` -> **MetaRegistry** (Acquires Write Lock) -> Stores Factory -> Returns `MetaKey`

> **Instantiation Flow (Runtime):**
> Game Logic with `MetaKey` -> `newMetaObject(key, owner)` -> **MetaRegistry** (Acquires Read Lock) -> Retrieves Factory -> Executes Factory -> Returns New Metadata Object

> **Deserialization Flow (Loading):**
> Serialized Data ("key") -> Deserializer -> `getMetaKeyForCodecKey("key")` -> **MetaRegistry** -> `PersistentMetaKey` -> Deserializer uses key to call `newMetaObject`

