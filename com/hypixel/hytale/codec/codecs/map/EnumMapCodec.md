---
description: Architectural reference for EnumMapCodec
---

# EnumMapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient Component

## Definition
```java
// Signature
public class EnumMapCodec<K extends Enum<K>, V> implements Codec<Map<K, V>>, WrappedCodec<V> {
```

## Architecture & Concepts
The EnumMapCodec is a specialized, high-performance component within the Hytale Codec subsystem responsible for serializing and deserializing Java Map instances that use an Enum as the key. It acts as a concrete implementation of the **Codec** interface, serving as a type adapter between the in-memory representation (**Map<K, V>**) and the on-disk or network representation (BSON or JSON).

Architecturally, this class embodies the **Composition over Inheritance** principle. It is a **WrappedCodec**, meaning it contains and delegates to another Codec instance for processing the map's values. This design allows for the flexible combination of codecs; for example, an EnumMapCodec can be configured with a StringCodec, an IntCodec, or another complex ObjectCodec to handle the serialization of its map values.

Its primary function is to enforce a strict contract for map keys. By leveraging Java Enums, it provides compile-time safety and runtime efficiency, often using the highly optimized **java.util.EnumMap** as its default backing implementation. The codec also supports configurable string formatting for enum keys (e.g., CAMEL_CASE vs. LEGACY_UPPERCASE), ensuring compatibility across different data formats or versions.

A key feature is its integration with the engine's schema generation system. The **toSchema** method produces a formal JSON Schema definition, making the data structure self-describing. This is critical for tooling, data validation, and automated content pipeline tasks.

## Lifecycle & Ownership
- **Creation:** An EnumMapCodec is not a managed singleton. It is instantiated directly via its constructor, typically during the application's bootstrap phase where the master **CodecRegistry** is configured. A new instance is created for each unique **Map<EnumType, ValueType>** combination required by the application.

- **Scope:** The lifecycle of an EnumMapCodec instance is tied to the component that created it, most often a central CodecRegistry. It is designed to be created once and reused for the entire application session for all serialization and deserialization operations of its specific map type.

- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It will be garbage collected when the CodecRegistry or other holding component is unloaded.

## Internal State & Concurrency
- **State:** The EnumMapCodec is **effectively immutable** after its initial configuration. Its core state, including the target enum class, the value codec, and the enum style, is set in the constructor and never modified. Internal caches for enum constants and their string representations are also populated at creation time for performance.

- **Thread Safety:** This class is **thread-safe** for all encoding and decoding operations. The **encode** and **decode** methods are pure functions with respect to the codec's state and can be safely invoked by multiple threads concurrently.

    **WARNING:** The builder-style method **documentKey** modifies internal state. It is not thread-safe and must only be called during the initial, single-threaded setup phase before the codec is made available to the rest of the system. Calling it after the codec is in active use will lead to race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bson, info) | Map<K, V> | O(N) | Deserializes a BsonDocument into a Map. N is the number of keys in the document. |
| encode(map, info) | BsonValue | O(N) | Serializes a Map into a BsonDocument. N is the number of entries in the map. |
| decodeJson(reader, info) | Map<K, V> | O(N) | Deserializes a raw JSON stream into a Map. N is the number of keys in the JSON object. |
| toSchema(context) | Schema | O(M) | Generates a JSON Schema definition for the map. M is the number of constants in the enum. |
| documentKey(key, doc) | EnumMapCodec | O(1) | Adds documentation for a specific enum key for use in schema generation. Not thread-safe. |
| getChildCodec() | Codec<V> | O(1) | Returns the underlying codec used for serializing the map's values. |

## Integration Patterns

### Standard Usage
The EnumMapCodec is typically instantiated once during system initialization and registered with a central registry. Direct invocation is rare; the serialization system dispatches to it automatically based on type.

```java
// During application bootstrap or static initialization

// 1. Define the Codec for the map's value type
Codec<BlockProperties> blockPropertiesCodec = ...;

// 2. Create the specialized EnumMapCodec
// This codec will handle Map<BlockFace, BlockProperties>
EnumMapCodec<BlockFace, BlockProperties> faceMapCodec = 
    new EnumMapCodec<>(BlockFace.class, blockPropertiesCodec);

// 3. (Optional) Add documentation for schema generation
faceMapCodec.documentKey(BlockFace.TOP, "Properties for the top face of the block.");
faceMapCodec.documentKey(BlockFace.SIDE, "Properties for the side faces of the block.");

// 4. Register with a central CodecRegistry (conceptual)
// codecRegistry.register(new TypeToken<Map<BlockFace, BlockProperties>>() {}, faceMapCodec);
```

### Anti-Patterns (Do NOT do this)
- **Per-Operation Instantiation:** Do not create a new EnumMapCodec for every serialization or deserialization call. This is highly inefficient as it re-calculates internal caches on every instantiation. Create once, reuse indefinitely.
- **Incorrect Key Type:** Attempting to use this codec for a Map whose keys are not Enums will result in runtime exceptions. It is purpose-built for Enum keys.
- **Post-Initialization Modification:** Do not call **documentKey** on a shared codec instance after the initialization phase. This can introduce race conditions if another thread is simultaneously using the codec to generate a schema.

## Data Pipeline
The EnumMapCodec acts as a transformation stage in the data serialization and deserialization pipeline.

> **Decoding Flow:**
> BsonDocument -> **EnumMapCodec.decode** -> (Iterates keys, looks up Enum constant, invokes child **ValueCodec.decode** for each value) -> In-Memory Map<K, V>

> **Encoding Flow:**
> In-Memory Map<K, V> -> **EnumMapCodec.encode** -> (Iterates entries, formats Enum key to string, invokes child **ValueCodec.encode** for each value) -> BsonDocument

