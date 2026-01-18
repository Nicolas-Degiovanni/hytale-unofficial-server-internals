---
description: Architectural reference for Object2DoubleMapCodec
---

# Object2DoubleMapCodec<T>

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Utility

## Definition
```java
// Signature
public class Object2DoubleMapCodec<T> implements Codec<Object2DoubleMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Object2DoubleMapCodec is a specialized, high-performance component within the Hytale Codec Framework responsible for serializing and deserializing `fastutil` Object2DoubleMap collections. It acts as a translator between the in-memory Java object representation and the on-disk or network-transportable formats of BSON and JSON.

Architecturally, this class embodies the **Strategy Pattern**. It is generic and does not inherently know how to handle the map's key type T. Instead, it delegates the responsibility of key serialization and deserialization to a separate, injected *keyCodec*. This compositional design makes the Object2DoubleMapCodec extremely reusable, as it can be configured to handle maps with any key type for which a corresponding Codec exists.

Furthermore, it utilizes a `Supplier` factory for map instantiation during decoding. This decouples the codec from any specific concrete map implementation (e.g., `Object2DoubleArrayMap`, `Object2DoubleOpenHashMap`), allowing the system integrator to choose the most memory or performance-efficient map type for a given use case.

## Lifecycle & Ownership
- **Creation:** Instances are created and configured during the bootstrap phase of the application, typically within a central `CodecRegistry` or a static definitions class. They are constructed with their dependencies (the key codec and map supplier) and are intended to be long-lived.
- **Scope:** An Object2DoubleMapCodec instance is stateless and designed to be reused for the entire application lifetime. Its scope is tied to the registry or context in which it is defined.
- **Destruction:** The object has no explicit destruction logic. It is eligible for garbage collection when its containing `CodecRegistry` or ClassLoader is unloaded, which typically only occurs at application shutdown.

## Internal State & Concurrency
- **State:** The Object2DoubleMapCodec is **effectively immutable**. Its internal fields, including the key codec and map supplier, are final and set only at construction. It maintains no state related to any specific encoding or decoding operation.
- **Thread Safety:** This class is **thread-safe**. Its immutable nature and the absence of shared, mutable state ensure that a single instance can be safely used by multiple threads concurrently to encode or decode different map objects without risk of interference or data corruption.

    **WARNING:** Thread safety is contingent on the injected `keyCodec` and `supplier` also being thread-safe. A stateful or improperly synchronized dependency will compromise the safety of this codec.

## API Surface
The public API provides the core serialization and deserialization contract required by the Codec framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Object2DoubleMap<T> | O(N) | Deserializes a BSON document into a new map instance. N is the number of entries in the document. |
| encode(Object2DoubleMap<T>, ExtraInfo) | BsonValue | O(N) | Serializes a map into a BSON document. N is the number of entries in the map. |
| decodeJson(RawJsonReader, ExtraInfo) | Object2DoubleMap<T> | O(N) | Deserializes a JSON object from a raw character stream into a new map. Optimized for low-level parsing. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema definition for validation and tooling, describing a map-like object. |
| getChildCodec() | Codec<T> | O(1) | Returns the underlying codec used for processing map keys. Fulfills the WrappedCodec contract. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke this class directly. Instead, it is instantiated once as part of a larger codec registry setup. The framework then dispatches to this implementation when it needs to process an Object2DoubleMap.

```java
// Example: Registering a codec for a map of ZoneId strings to double values.

// 1. Define the codec with its dependencies.
Codec<Object2DoubleMap<String>> stringToDoubleMapCodec = new Object2DoubleMapCodec<>(
    Codecs.STRING,              // The strategy for handling the String key
    Object2DoubleArrayMap::new, // The factory for creating new maps on decode
    true                        // Deserialized maps will be unmodifiable
);

// 2. Register it in the central codec system (conceptual).
CodecRegistry.register(Object2DoubleMap.class, stringToDoubleMapCodec);

// 3. The system can now transparently handle this map type.
Object2DoubleMap<String> myMap = ...;
BsonValue bson = Codecs.encode(myMap); // The registry dispatches to our codec.
```

### Anti-Patterns (Do NOT do this)
- **Stateful Supplier:** Never provide a `Supplier` that returns a pre-existing or shared map instance. The supplier **must** return a new, empty map on every invocation to prevent data corruption and severe concurrency bugs.
- **Incorrect Key Codec:** The provided `keyCodec` must serialize type T to a format that is a valid BSON or JSON map key (i.e., a string). Providing a codec that serializes to a number or a complex object will result in a runtime exception.
- **Per-Operation Instantiation:** Do not create a new Object2DoubleMapCodec for each serialization task. This is inefficient and defeats the purpose of the stateless, reusable design. Define instances once and reuse them.

## Data Pipeline
The Object2DoubleMapCodec is a specific processing stage in the overall serialization pipeline.

> **Encoding Flow:**
> In-Memory `Object2DoubleMap<T>` -> **Object2DoubleMapCodec**.encode() -> `BsonDocument` / JSON Stream

> **Decoding Flow:**
> `BsonDocument` / JSON Stream -> **Object2DoubleMapCodec**.decode() -> In-Memory `Object2DoubleMap<T>`

