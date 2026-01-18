---
description: Architectural reference for Short2ObjectMapCodec
---

# Short2ObjectMapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient

## Definition
```java
// Signature
public class Short2ObjectMapCodec<T> implements Codec<Short2ObjectMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Short2ObjectMapCodec is a specialized, high-performance component within the Hytale serialization framework. Its primary function is to translate between the in-memory representation of a `fastutil` Short2ObjectMap and a serialized format, specifically BSON and a custom raw JSON stream.

This class acts as a bridge, enabling the engine to use memory-efficient map implementations with primitive keys (`short`) while maintaining compatibility with standard, string-keyed serialization formats. It embodies the **Composition over Inheritance** principle by wrapping a generic `Codec<T>` for its values. This design allows it to serialize a map of shorts to *any* type, provided a codec for that type exists.

As a `WrappedCodec`, it exposes its internal value codec. This is critical for higher-level systems, such as the schema generator, which need to introspect the full structure of a data type to build a complete definition.

## Lifecycle & Ownership
- **Creation:** An instance is created via its constructor, typically by a central `CodecRegistry` or a factory responsible for assembling the codec graph. The caller must provide a `Codec` for the map's value type and a `Supplier` to construct new map instances during decoding.
- **Scope:** The lifecycle of a Short2ObjectMapCodec instance is tied to the component that created it, usually a registry. It is designed to be stateless and reusable, persisting for the entire application session once registered.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It will be destroyed when the `CodecRegistry` or other owning component is garbage collected.

## Internal State & Concurrency
- **State:** The class is effectively immutable. Its internal fields—`valueCodec`, `supplier`, and `unmodifiable`—are final and set during construction. It does not cache any data between calls; each `encode` or `decode` operation is self-contained.
- **Thread Safety:** This class is thread-safe. Its immutable state and lack of side effects ensure that a single instance can be safely used by multiple threads concurrently to encode or decode different map instances.

**WARNING:** Thread safety is contingent on the provided `valueCodec` and `supplier` also being thread-safe. Passing non-thread-safe dependencies will violate the concurrency contract.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Short2ObjectMap | O(N) | Deserializes a BSON document into a map. N is the number of entries. Throws CodecException on failure. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes a map into a BSON document. N is the number of entries. |
| decodeJson(reader, extraInfo) | Short2ObjectMap | O(N) | Deserializes a JSON object from a raw stream into a map. N is the number of entries. |
| getChildCodec() | Codec | O(1) | Returns the wrapped codec used for the map's values. |
| toSchema(context) | Schema | O(1) | Generates a data schema definition for this map type. |

## Integration Patterns

### Standard Usage
This codec is not intended for direct use by most developers. It is designed to be registered with a central codec management system, which then invokes it automatically when a field of type `Short2ObjectMap` is encountered.

```java
// Hypothetical registration with a CodecRegistry
Codec<BlockState> blockStateCodec = ...;
Supplier<Short2ObjectMap<BlockState>> mapSupplier = Short2ObjectOpenHashMap::new;

// The registry would construct and store the codec
registry.register(
    new Short2ObjectMapCodec<>(blockStateCodec, mapSupplier)
);

// Later, another system uses the registry to process data
ChunkData data = registry.decode(ChunkData.class, bson);
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Avoid calling `decode` or `encode` directly. Rely on a higher-level abstraction like a `CodecRegistry` to ensure proper context and error handling are applied.
- **Stateful Dependencies:** Do not provide a `Supplier` or `Codec` that relies on mutable, non-thread-safe state. This will break concurrency guarantees and lead to unpredictable behavior under load.
- **Incorrect Key Parsing:** The codec expects map keys in the serialized format to be valid string representations of a `short`. Providing non-numeric or out-of-range keys will result in a `CodecException`.

## Data Pipeline

### Decoding Flow
The pipeline transforms a serialized document into a live, in-memory `fastutil` map.

> Flow:
> BsonDocument -> **Short2ObjectMapCodec.decode** -> (For each entry: Parse String key to `short`, Delegate BsonValue to `valueCodec`) -> Populated Short2ObjectMap

### Encoding Flow
The pipeline transforms an in-memory `fastutil` map into a serialized BSON document.

> Flow:
> Short2ObjectMap -> **Short2ObjectMapCodec.encode** -> (For each entry: Convert `short` key to String, Delegate value to `valueCodec`) -> Populated BsonDocument

