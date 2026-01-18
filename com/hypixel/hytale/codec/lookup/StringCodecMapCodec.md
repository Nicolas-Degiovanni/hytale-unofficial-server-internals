---
description: Architectural reference for StringCodecMapCodec
---

# StringCodecMapCodec

**Package:** com.hypixel.hytale.codec.lookup
**Type:** Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class StringCodecMapCodec<T, C extends Codec<? extends T>> extends ACodecMapCodec<String, T, C> {
```

## Architecture & Concepts

The StringCodecMapCodec is a foundational component of the Hytale serialization framework. It functions as a high-performance, polymorphic dispatcher for JSON deserialization. Its primary role is to inspect an incoming JSON stream, identify a specific string key, and delegate the full deserialization process to a specialized Codec registered to handle that key.

This class is the architectural solution for deserializing collections of heterogeneous types that share a common base class. For example, a list of different *Block* types in a world file can be represented in JSON with a type identifier field. The StringCodecMapCodec reads this identifier and selects the correct *StoneBlockCodec*, *DirtBlockCodec*, or other appropriate handler.

The key architectural feature is its direct integration with the raw JSON stream via the internal **StringTreeMap**. This specialized data structure avoids the performance overhead of fully parsing the identifier string into a Java String object before performing a lookup. Instead, it matches characters directly from the reader's buffer, making it exceptionally efficient for hot-path deserialization routines like chunk loading and network packet processing.

## Lifecycle & Ownership

-   **Creation:** As an abstract class, it is never instantiated directly. Concrete subclasses (e.g., a BlockTypeCodecMap) are instantiated once during the engine's bootstrap sequence, typically by a central service responsible for codec registration.
-   **Scope:** An instance of a StringCodecMapCodec is a long-lived, session-scoped object. It is configured at startup and persists for the entire application lifetime, serving all deserialization requests for its designated type.
-   **Destruction:** The object is not explicitly destroyed. It is subject to standard garbage collection when the application terminates and the class loaders are unloaded.

## Internal State & Concurrency

-   **State:** The StringCodecMapCodec is a stateful and highly mutable component. Its core state is the mapping of string identifiers to Codec instances held within the internal StringTreeMap. This mapping can be modified at runtime via the register and remove methods, allowing for dynamic extensibility (e.g., by mods).

-   **Thread Safety:** This component is designed for concurrent access but contains a critical implementation flaw.
    -   **Read Operations:** The primary `decodeJson` method is protected by a `StampedLock` read lock. This allows multiple threads to safely perform deserialization concurrently, provided no write operations are occurring.
    -   **Write Operations:** The `register` and `remove` methods are intended to modify the internal state. However, they incorrectly acquire a `readLock` instead of a `writeLock`.

    **WARNING:** This flawed locking strategy creates a severe race condition. Modifying the codec map via `register` or `remove` while other threads are calling `decodeJson` can lead to `ConcurrentModificationException`, corrupted internal state, or other unpredictable runtime failures. All registration and removal operations **MUST** be performed during a single-threaded, controlled initialization phase before the codec is used for active deserialization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(priority, id, aClass, codec) | StringCodecMapCodec | O(log N) | Adds a new Codec to the internal map, associating it with a string ID. **Not thread-safe for concurrent writes.** |
| remove(aClass) | void | O(log N) | Removes a Codec from the map based on its class type. **Not thread-safe for concurrent writes.** |
| decodeJson(reader, extraInfo) | T | O(M + log N) | The primary deserialization entry point. Reads a string key from the stream and delegates to the corresponding registered Codec. M is the length of the key string. |

## Integration Patterns

### Standard Usage

A developer should extend this class to create a specific codec dispatcher. All child codecs are then registered with the instance during a safe, single-threaded bootstrap phase. The instance is then passed to higher-level systems that manage serialization.

```java
// 1. Define a concrete implementation for a specific object hierarchy.
public class EntityCodecMap extends StringCodecMapCodec<Entity, Codec<? extends Entity>> {
    public EntityCodecMap() {
        super("entityType", true); // The key in JSON is "entityType"
    }
}

// 2. During engine initialization, create an instance and register all known types.
EntityCodecMap entityCodecs = new EntityCodecMap();
entityCodecs.register(Priority.NORMAL, "hytale:creeper", Creeper.class, new CreeperCodec());
entityCodecs.register(Priority.NORMAL, "hytale:player", Player.class, new PlayerCodec());

// 3. The engine will then use this instance to decode data from a network packet or file.
// Direct invocation is rare; it is typically used by a parent parser.
Entity entity = someJsonParser.decode(json, entityCodecs);
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Never call `register` or `remove` after the initialization phase while the engine might be actively decoding data on other threads. The flawed locking makes this operation extremely hazardous.
-   **Per-Operation Instantiation:** Do not create a new instance of a StringCodecMapCodec for each deserialization task. It is a heavyweight, stateful object designed to be a long-lived singleton.

## Data Pipeline

The flow of data during a deserialization operation is a multi-stage dispatch process, orchestrated by this class.

> Flow:
> Raw JSON Stream -> **StringCodecMapCodec.decodeJson** -> Efficient key lookup via StringTreeMap -> Selects specific `Codec<T>` -> `SpecificCodec.decodeJson` -> Fully instantiated `Object<T>`

