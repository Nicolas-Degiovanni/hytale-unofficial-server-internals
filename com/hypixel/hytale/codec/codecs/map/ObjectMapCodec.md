---
description: Architectural reference for ObjectMapCodec
---

# ObjectMapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class ObjectMapCodec<K, V, M extends Map<K, V>> implements Codec<Map<K, V>>, WrappedCodec<V> {
```

## Architecture & Concepts

The ObjectMapCodec is a specialized component within the Hytale serialization framework responsible for the bidirectional conversion between a Java Map and its BSON or JSON object representation.

Its primary architectural purpose is to serve as a generic, configurable bridge for maps where the keys are not simple strings. Standard JSON and BSON object formats mandate string-based keys. This codec overcomes that limitation by composing two user-provided functions, one to convert a generic key type K to a String for encoding, and another to convert a String back to type K during decoding.

This component follows a composite design pattern. It is a *wrapped codec*, meaning it does not handle the serialization of the map's values (type V) itself. Instead, it holds a reference to a child Codec instance and delegates all value-level serialization and deserialization tasks to it. This separation of concerns allows for complex, nested data structures to be defined declaratively.

> **Warning: Deprecated Component**
> This class is marked as deprecated. It should not be used for new development. It remains in the codebase for backward compatibility with legacy data formats. Newer systems should utilize alternative, more robust map serialization strategies provided by the framework.

## Lifecycle & Ownership

-   **Creation:** ObjectMapCodec is not managed by a central service registry. It is instantiated directly by the developer or a higher-level configuration system when a specific map serialization rule is required. Construction requires providing a value codec, a map supplier, and key transformation functions.
-   **Scope:** Transient. An instance of ObjectMapCodec is stateless and its lifetime is typically bound to the configuration object or parent codec that created it. It does not persist beyond the application's runtime.
-   **Destruction:** The object is subject to standard Java garbage collection once it is no longer referenced. No explicit cleanup or destruction methods are required.

## Internal State & Concurrency

-   **State:** Immutable. The internal fields of ObjectMapCodec, including the child codec, map supplier, and key functions, are set at construction and are not modified thereafter. The class holds no per-operation state.
-   **Thread Safety:** This class is inherently thread-safe. Its methods operate exclusively on local variables and the provided arguments. Concurrency safety is therefore dependent on the guarantees of its dependencies: the child codec, the map supplier, and the key transformation functions must also be thread-safe for the entire operation to be safe in a multithreaded environment.

## API Surface

The public contract is defined by the Codec and WrappedCodec interfaces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Map | O(N) | Deserializes a BSON document into a new Map instance. N is the number of key-value pairs. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes a Map into a BSON document. N is the number of key-value pairs. |
| decodeJson(reader, extraInfo) | Map | O(N) | Deserializes a JSON object from a low-level stream into a new Map instance. |
| getChildCodec() | Codec | O(1) | Returns the wrapped codec responsible for serializing the map's values. |
| toSchema(context) | Schema | O(1) | Generates a data schema for the map structure, defining keys as strings and values by the child codec's schema. |

## Integration Patterns

### Standard Usage

This codec is designed to be composed. A developer defines the complete serialization behavior by providing all dependencies at construction time.

```java
// Example: A codec for a Map<WorldZone, ZoneData>
// where WorldZone is an enum and ZoneData has its own codec.

Codec<ZoneData> zoneDataCodec = new ZoneDataCodec();
Codec<Map<WorldZone, ZoneData>> worldZoneMapCodec = new ObjectMapCodec<>(
    zoneDataCodec,                 // The codec for the map's values
    HashMap::new,                  // A supplier for the map implementation
    WorldZone::name,               // Function to convert key (enum) to string
    WorldZone::valueOf,            // Function to convert string back to key
    false                          // Create a mutable map
);

// This codec can now be used to serialize and deserialize the entire map.
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Code:** The primary anti-pattern is using this deprecated class. Refer to current framework best practices for the recommended replacement.
-   **Asymmetric Key Functions:** Providing `keyToString` and `stringToKey` functions that are not perfect inverses of each other. This will lead to data corruption on a round-trip serialization or throw an exception during decoding. For any key `k`, `stringToKey.apply(keyToString.apply(k))` must equal `k`.
-   **Shared Map Supplier:** Providing a supplier that returns a pre-existing or shared map instance instead of a new one (e.g., `() -> mySharedMap` instead of `HashMap::new`). This will break thread safety and cause unpredictable behavior, as multiple decode operations will attempt to write to the same shared map instance.

## Data Pipeline

The flow of data through this component is straightforward, acting as a dispatcher for keys and values.

**Decoding Flow:**
> BSON Document -> **ObjectMapCodec.decode** -> (For each entry) -> `stringToKey(entry.key)` + `valueCodec.decode(entry.value)` -> Populated Java Map

**Encoding Flow:**
> Java Map -> **ObjectMapCodec.encode** -> (For each entry) -> `keyToString(entry.key)` + `valueCodec.encode(entry.value)` -> BSON Document

