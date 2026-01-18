---
description: Architectural reference for MergedEnumMapCodec
---

# MergedEnumMapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient

## Definition
```java
// Signature
public class MergedEnumMapCodec<K extends Enum<K>, V, M extends Enum<M>> implements Codec<Map<K, V>>, WrappedCodec<V> {
```

## Architecture & Concepts
The MergedEnumMapCodec is a specialized, high-performance component within the Hytale serialization framework. Its primary architectural role is to provide a powerful data-aliasing mechanism for configuration files and network packets. It deserializes a map-like data structure (BSON or JSON) into a Java EnumMap, but with a critical enhancement: it can interpret special "group" keys that expand into multiple entries in the final map.

This is achieved by operating on two distinct Enum types:
1.  **Primary Enum (K):** The target Enum type for the keys in the final in-memory EnumMap.
2.  **Merge Enum (M):** A secondary Enum type whose constants act as aliases or groups for one or more constants from the primary Enum.

When a key from the source data matches a constant in the Merge Enum, the codec uses a provided **unmergeFunction** to determine the corresponding set of Primary Enum keys. The value associated with the group key is then applied to all of these expanded keys. If a key collision occurs during this expansion (i.e., a key is specified both directly and via a group), a provided **mergeResultFunction** resolves the conflict.

This pattern is essential for creating data-driven assets that are both concise for human authors and explicit for the game engine. For example, a single key like *ALL_ARMOR* in a JSON file could be expanded into *HELMET*, *CHESTPLATE*, *LEGGINGS*, and *BOOTS* in the resulting EnumMap, reducing verbosity and preventing configuration errors.

## Lifecycle & Ownership
-   **Creation:** Instances of MergedEnumMapCodec are not intended for direct instantiation within general application logic. They are constructed during the static definition phase of a parent codec, typically using a builder pattern like that of ObjectCodec. The codec system itself manages the instantiation based on declarative schema definitions.
-   **Scope:** A MergedEnumMapCodec instance is stateless and designed for reuse. It persists for the entire application session, held as a final field within the larger codec that owns it.
-   **Destruction:** The object is managed by the Java garbage collector. It has no external resources to release and is cleaned up when its owning codec is no longer reachable.

## Internal State & Concurrency
-   **State:** The MergedEnumMapCodec is **effectively immutable**. All of its configuration, including the child codec and functional interfaces, is provided via the constructor and stored in final fields. To optimize performance, it pre-computes and caches the string representations of all enum keys from both K and M during construction.
-   **Thread Safety:** This class is **fully thread-safe**. Its immutable nature ensures that concurrent calls to encode or decode cannot interfere with each other. Each call to a decode method uses the provided `Supplier` to create a new, distinct map instance, preventing any state from leaking between deserialization operations on different threads. This design is critical for the engine's parallel asset loading and network processing systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Map<K, V> | O(N) | Deserializes a BsonDocument into an EnumMap, performing key expansion and merging. N is the number of keys in the source document. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes an EnumMap into a BsonDocument. This is a direct mapping and does not perform key merging. N is the size of the map. |
| decodeJson(reader, extraInfo) | Map<K, V> | O(N) | Deserializes a JSON object from a stream into an EnumMap. Functionally equivalent to the BSON version. |
| toSchema(context) | Schema | O(K+M) | Generates a JSON Schema definition for tooling and validation. K and M are the number of constants in the respective enums. |
| getChildCodec() | Codec<V> | O(1) | Returns the underlying codec used for serializing and deserializing the map's values. |

## Integration Patterns

### Standard Usage
This codec is not used in isolation. It is integrated into the definition of a larger codec for a configuration object or data transfer object.

```java
// Example: Defining a codec for a component that uses permission groups.
// The JSON can use "ADMIN" to grant a set of specific permissions.
public static final Codec<ComponentPermissions> CODEC = ObjectCodec.create(
    ComponentPermissions::new,
    new MergedEnumMapCodec<>(
        Permission.class,         // The target enum (e.g., READ, WRITE)
        PermissionGroup.class,    // The merge enum (e.g., ADMIN, GUEST)
        PermissionGroup::getPermissions, // Function M -> K[]
        (existingValue, newValue) -> newValue, // Merge strategy: new value wins
        Codec.BOOL                // Codec for the map's value
    ).fieldOf("permissions", ComponentPermissions::getPermissions)
);
```

### Anti-Patterns (Do NOT do this)
-   **Instantiation in Game Logic:** Never call `new MergedEnumMapCodec(...)` inside a game loop or in response to an event. Codecs must be defined statically and reused.
-   **Stateful Lambdas:** The `unmergeFunction` and `mergeResultFunction` must be pure, stateless functions. Introducing I/O, network calls, or mutable state into these functions will lead to severe performance degradation and unpredictable, non-deterministic behavior.
-   **Reused Supplier Target:** The `Supplier<EnumMap<K, V>>` provided to the constructor **must** create a new map instance on every call. Reusing a single map instance will cause catastrophic state corruption across concurrent decode operations.

## Data Pipeline
The flow of data through this component is different for reading (decoding) and writing (encoding).

> **Flow (Decode):**
> BSON/JSON Document → **MergedEnumMapCodec.decode()** → For each key-value pair:
> 1.  Lookup key in Primary Enum (K) and Merge Enum (M).
> 2.  If key is in M, execute **unmergeFunction** to get an array of K keys.
> 3.  Decode value using the child `Codec<V>`.
> 4.  For each target K key, put the decoded value into the new map.
> 5.  If a key already exists, execute **mergeResultFunction** to resolve the conflict.
> → Final `EnumMap<K, V>`

> **Flow (Encode):**
> `EnumMap<K, V>` → **MergedEnumMapCodec.encode()** → For each entry in the map:
> 1.  Convert the Enum key (K) to its string representation.
> 2.  Encode the value using the child `Codec<V>`.
> 3.  Put the string key and encoded value into a new BsonDocument.
> → BsonDocument

