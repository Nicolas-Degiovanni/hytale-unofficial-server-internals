---
description: Architectural reference for MapKeyMapCodec
---

# MapKeyMapCodec<V>

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Stateful Factory

## Definition
```java
// Signature
public class MapKeyMapCodec<V> extends AMapProvidedMapCodec<Class<? extends V>, V, Codec<V>, MapKeyMapCodec.TypeMap<V>> {
```

## Architecture & Concepts
The MapKeyMapCodec is a sophisticated serialization system designed for handling polymorphic data structures. It acts as a central, dynamically-updatable registry that maps string identifiers from a data source (e.g., BSON, JSON) to specific Java Class types and their corresponding serialization logic (Codecs).

Its primary role is to enable the engine to deserialize collections of objects where the concrete types are not known at compile time. For example, a list of entity components, a set of network events, or a collection of status effects can all be managed by this system. The codec reads a string key, uses it to look up the correct Class and Codec from its internal registry, and then deserializes the associated data into a strongly-typed Java object.

A key architectural feature is its support for **runtime type registration and data upgrading**. The system is designed to handle cases where data is loaded *before* the corresponding Codec has been registered, a common scenario in modding or asynchronous content loading. Data associated with an unknown type identifier is preserved in a raw, unprocessed state. If a Codec for that identifier is later registered, the system can automatically "upgrade" the raw data into a fully-typed object within all existing data maps. This is managed through a global weak-reference tracking system for all live map instances.

The concrete data structure produced by this codec is the inner class MapKeyMapCodec.TypeMap, which acts as the runtime container for the deserialized, type-keyed objects.

## Lifecycle & Ownership
The lifecycle of this class is complex due to its management of shared, static, global state.

-   **Creation:** Instances are typically created once by a higher-level manager system during application bootstrap. For example, an `EntityComponentManager` would create a `MapKeyMapCodec<Component>` to handle the serialization of all component types.

-   **Scope:** While an individual `MapKeyMapCodec` instance can be short-lived, the type registrations it performs are **global and static**. They are stored in static maps and persist for the entire lifetime of the Java Virtual Machine. This means registrations from one codec instance are visible to all others. A static daemon thread, "MapKeyMapCodec", is also spawned on class load and persists for the application's lifetime to perform background cleanup of garbage-collected `TypeMap` instances.

-   **Destruction:** The `MapKeyMapCodec` object itself is eligible for garbage collection when it is no longer referenced. However, this **does not** automatically unregister the types it defined. Type mappings must be explicitly removed via the `unregister` method if they need to be cleaned up.

## Internal State & Concurrency
-   **State:** The class manages a highly mutable, shared, static state consisting of three core maps: `idToClass`, `classToId`, and the `codecProvider` inherited from its parent. This state represents the global registry of all known polymorphic types. The `TypeMap` instances it produces are also mutable, containing both deserialized objects and a cache of raw BsonValue objects for any unrecognized keys.

-   **Thread Safety:** This class is explicitly designed to be thread-safe and implements a sophisticated locking strategy.
    -   A single, static `StampedLock` governs all access to the type registry.
    -   **Write Operations:** Methods that modify the registry, such as `register` and `unregister`, acquire a full `writeLock`. This ensures that the type system remains consistent during modification, but it is a globally blocking operation.
    -   **Read Operations:** Methods that access the data within a `TypeMap`, such as `get` or `put`, acquire a `readLock`. This allows for highly concurrent reads from multiple threads, as long as no write operation is in progress.
    -   The use of `ConcurrentHashMap` for internal maps provides an additional layer of thread safety for the collections themselves.

## API Surface
The public API is focused on registry management and map creation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(Class, String, Codec) | void | O(N) | Registers a new type, its ID, and its codec. **Warning:** Complexity is linear to the number of active `TypeMap` instances (N) due to the live-upgrade scan. |
| unregister(Class) | void | O(N) | Removes a type from the global registry. **Warning:** Complexity is linear to the number of active `TypeMap` instances (N) due to the live-downgrade scan. |
| createMap() | TypeMap<V> | O(1) | Factory method to create a new, empty, mutable `TypeMap` instance that is linked to this codec's configuration. |
| getKeyForId(String) | Class<? extends V> | O(1) | Retrieves the Class associated with a given string identifier. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a codec, register all necessary types during an initialization phase, and then use the codec to create and populate map instances.

```java
// 1. Create the codec instance during system initialization
MapKeyMapCodec<GameEvent> eventCodec = new MapKeyMapCodec<>();

// 2. Register all known event types
eventCodec.register(PlayerJoinEvent.class, "hytale:player_join", new PlayerJoinEventCodec());
eventCodec.register(BlockBreakEvent.class, "hytale:block_break", new BlockBreakEventCodec());

// 3. Use the codec to create a map for holding event data
TypeMap<GameEvent> eventData = eventCodec.createMap();

// 4. Populate the map at runtime
eventData.put(PlayerJoinEvent.class, new PlayerJoinEvent("Player1"));
```

### Anti-Patterns (Do NOT do this)
-   **Frequent Registration:** Do not call `register` or `unregister` in performance-critical loops (e.g., per-frame, per-network-packet). These methods acquire a global write lock and iterate over all active maps, which can cause significant contention and performance degradation. Type registration should be a one-time setup operation.
-   **Ignoring Global State:** Do not create multiple `MapKeyMapCodec` instances for the same base type `V` with overlapping string IDs or classes. All instances share the same static registry, and attempting to register a duplicate ID or Class will throw an `IllegalArgumentException`. This can be a source of subtle bugs if different systems are unaware of each other's registrations.

## Data Pipeline
The codec operates as a bidirectional bridge between raw BSON data and strongly-typed Java objects.

> **Decoding Flow:**
> BSON Document -> `AMapProvidedMapCodec` (iterates fields) -> For each field, `MapKeyMapCodec` uses the field name as an ID -> Looks up `Class` and `Codec` in static registry -> **If found:** `Codec.decode()` is called -> `(Class, Object)` pair is put into `TypeMap`. -> **If not found:** `(String, BsonValue)` pair is stored in `TypeMap.unknownValues`.

> **Encoding Flow:**
> `TypeMap` instance -> `AMapProvidedMapCodec` (iterates entries) -> For each `(Class, Object)` entry, `MapKeyMapCodec` looks up the `String ID` and `Codec` -> `Codec.encode()` is called -> `(String ID, BsonValue)` is written to the BSON Document. -> Any entries in `TypeMap.unknownValues` are also written back to the document, preserving them.

