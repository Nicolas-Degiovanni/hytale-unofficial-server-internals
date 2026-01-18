---
description: Architectural reference for MapCodec
---

# MapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient

## Definition
```java
// Signature
public class MapCodec<V, M extends Map<String, V>> implements Codec<Map<String, V>>, WrappedCodec<V> {
```

## Architecture & Concepts
The MapCodec is a fundamental, composite component within the Hytale serialization framework. It provides a generic mechanism for encoding and decoding Java Map instances where keys are exclusively Strings. Its primary role is to bridge the gap between in-memory map data structures and their serialized representations in BSON or JSON.

Architecturally, MapCodec embodies the **Composite Pattern**. It does not handle the serialization of the map's values itself. Instead, it wraps a child Codec responsible for the value type V. This design promotes reusability and separation of concerns. The MapCodec manages the structure of the map—the key-value pairs and object delimiters ({})—while delegating the complex task of value serialization to the specialized child codec.

A key feature is its use of a `Supplier<M>` for map instantiation during decoding. This allows developers to control the concrete Map implementation (e.g., HashMap, LinkedHashMap, or a high-performance alternative like Object2ObjectOpenHashMap), decoupling the codec from specific collection types.

### Lifecycle & Ownership
-   **Creation:** Instances are created directly via their constructor, typically during the static initialization of a larger, more complex Codec that contains a map field. They are building blocks, not managed services. A common pattern is to define them as static final fields for reuse, as seen with `STRING_HASH_MAP_CODEC`.
-   **Scope:** The lifecycle of a MapCodec instance is tied to the object that holds its reference. For static codecs, this scope is the lifetime of the application. For transient instances, they are eligible for garbage collection once the parent codec or configuration object is no longer referenced.
-   **Destruction:** There is no explicit destruction or cleanup method. Management is handled entirely by the Java Garbage Collector.

## Internal State & Concurrency
-   **State:** A MapCodec instance is **Immutable**. Its internal state, consisting of the child `codec`, the map `supplier`, and the `unmodifiable` flag, is established at construction and never changes.
-   **Thread Safety:** The class is inherently **Thread-Safe**. Its immutable nature ensures that multiple threads can safely call its encode and decode methods concurrently without risk of data corruption. All operations are self-contained and do not modify the codec's internal state. The stateful, per-operation context is passed in via the `ExtraInfo` parameter, which is the responsibility of the calling thread to manage.

## API Surface
The public API is focused on the core serialization and deserialization contract defined by the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Map | O(N) | Deserializes a BSON document into a Java Map. N is the number of keys in the document. Throws CodecException on failure. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes a Java Map into a BSON document. N is the number of entries in the map. Skips null or empty values. |
| decodeJson(reader, extraInfo) | Map | O(N) | Deserializes a JSON object from a character stream into a Java Map. N is the number of keys in the object. |
| toSchema(context) | Schema | O(1) | Generates a schema definition for the map, describing it as an object with arbitrary string keys and values defined by the child codec's schema. |
| getChildCodec() | Codec | O(1) | Returns the wrapped codec responsible for serializing the map's values. |

## Integration Patterns

### Standard Usage
A MapCodec is rarely used in isolation. It is typically composed with other codecs to define the serialization rules for a complex object.

```java
// Define a codec for a map of game settings, from string keys to integer values.
// We supply a standard HashMap constructor for instantiation during decoding.
Codec<Map<String, Integer>> SETTINGS_MAP_CODEC = new MapCodec<>(Codec.INT, HashMap::new);

// This map codec would then be used as part of a larger object codec.
public class PlayerProfile {
    public static final Codec<PlayerProfile> CODEC = ObjectCodec.create(
        instance -> instance
            .field("name", Codec.STRING, profile -> profile.name)
            .field("settings", SETTINGS_MAP_CODEC, profile -> profile.settings)
    );

    private String name;
    private Map<String, Integer> settings;
}
```

### Anti-Patterns (Do NOT do this)
-   **Mismatched Supplier:** Providing a `Supplier` for a fixed-size or immutable map implementation will cause runtime exceptions during decoding, as the `put` method will fail. The supplied map must be mutable during the decoding process.
-   **Ignoring Immutability:** The `unmodifiable` flag is a critical safety feature. If you configure the codec to return mutable maps (`unmodifiable = false`), do not share these map instances across threads or systems without proper synchronization, as this can lead to race conditions.
-   **Incorrect Child Codec:** Passing a `Codec<String>` to a `MapCodec<Integer, ...>` will result in a `ClassCastException` or `CodecException` at runtime. The type parameterization must be consistent.

## Data Pipeline
The MapCodec acts as a structural processor in the data pipeline, orchestrating the flow between a serialized object and an in-memory Map.

**Decoding Pipeline**
> Flow:
> BSON Document -> **MapCodec.decode()** -> Iterate entries -> For each entry, invoke **ChildCodec.decode()** on the BsonValue -> Populate new Map instance -> Return final Map

**Encoding Pipeline**
> Flow:
> Java Map -> **MapCodec.encode()** -> Iterate entries -> For each entry, invoke **ChildCodec.encode()** on the value -> Populate new BsonDocument -> Return final BsonDocument

