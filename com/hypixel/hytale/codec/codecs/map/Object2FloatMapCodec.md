---
description: Architectural reference for Object2FloatMapCodec
---

# Object2FloatMapCodec<T>

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient Utility

## Definition
```java
// Signature
public class Object2FloatMapCodec<T> implements Codec<Object2FloatMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Object2FloatMapCodec is a specialized, high-performance component within the Hytale serialization framework. Its primary function is to translate between in-memory `Object2FloatMap` collections and their serialized representations in BSON or JSON.

This class embodies the **Composite Codec** pattern. It is not a monolithic serializer; instead, it manages the *structure* of the map (the key-value pairs) while delegating the serialization of the map's keys to a separate, "wrapped" child codec. This design decouples the logic for handling map structures from the logic for handling specific key types, promoting modularity and reuse. For example, the same Object2FloatMapCodec can be configured to handle a map of strings to floats or a map of custom entity identifiers to floats simply by providing a different child codec.

The implementation is heavily optimized, using the fastutil library for memory-efficient map representations and providing a direct-to-JSON decoding path (`decodeJson`) that bypasses intermediate object creation for maximum performance, which is critical for parsing configuration files. The `WrappedCodec` interface serves as a marker, allowing the codec system to introspect and understand that this codec is a container for another, which is essential for schema generation and system-wide validation.

## Lifecycle & Ownership
- **Creation:** Instances are not intended to be created directly by developers. They are constructed and configured by a central codec registry or via static factory methods on the primary `Codec` interface during application bootstrap. The configuration includes providing the child codec for the key and a supplier for the map implementation.
- **Scope:** An instance of Object2FloatMapCodec is typically stateless and lives for the entire application session. It is registered once for a specific map type (e.g., `Object2FloatMap<String>`) and reused for all serialization and deserialization operations of that type.
- **Destruction:** The object is garbage collected when the parent codec registry is discarded, which normally occurs only on application shutdown.

## Internal State & Concurrency
- **State:** This class is **immutable and stateless**. Its configuration, including the child codec and map supplier, is provided at construction and cannot be changed. It does not cache data or maintain any state between method invocations.
- **Thread Safety:** The Object2FloatMapCodec is **fully thread-safe**. Due to its stateless and immutable nature, a single instance can be safely shared across multiple threads to perform concurrent encoding and decoding operations without locks or synchronization.

## API Surface
The public contract is defined by the `Codec` interface. The following are the primary entry points for serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Object2FloatMap<T> | O(N) | Deserializes a BSON document into a new map instance. N is the number of keys. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes a map into a BSON document. N is the number of keys. |
| decodeJson(reader, extraInfo) | Object2FloatMap<T> | O(N) | Deserializes a JSON object from a raw stream into a new map. Highly optimized. |
| toSchema(context) | Schema | O(1) | Generates a data schema representing a map of a specific key type to a float. |

## Integration Patterns

### Standard Usage
Developers should never instantiate this class directly. It is used implicitly by the codec system when a field of type `Object2FloatMap` is encountered. The codec is typically defined declaratively as part of a larger object's codec.

```java
// Example of defining a codec for an object that USES Object2FloatMapCodec internally
// The system will automatically select Object2FloatMapCodec for the 'weights' field.

public static final Codec<MyConfig> CODEC = ObjectCodec.create(
    MyConfig::new,
    Codec.forString().field("name", MyConfig::getName),
    Codec.map(Codec.forString(), Object2FloatOpenHashMap::new).field("weights", MyConfig::getWeights)
);

// Usage:
MyConfig config = Codec.decode(CODEC, bsonData);
BsonValue bson = Codec.encode(CODEC, config);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new Object2FloatMapCodec(...)`. This bypasses the framework's registration and caching mechanisms and can lead to inconsistent serialization behavior. Always use the provided `Codec.map(...)` factory methods.
- **Reusing Map Instances:** The `Supplier` provided during construction **must** create a new map instance on every call. Returning a shared or singleton map will cause severe data corruption and race conditions during decoding.

## Data Pipeline
The Object2FloatMapCodec acts as a structural bridge in the data serialization pipeline, delegating type-specific logic to its child codec.

**Encoding Flow**
> In-Memory `Object2FloatMap<T>` -> **Object2FloatMapCodec** -> Child `Codec<T>` (for each key) -> `BsonDocument` -> Network or Disk

**Decoding Flow**
> Network or Disk -> `BsonDocument` -> **Object2FloatMapCodec** -> Child `Codec<T>` (for each key) -> In-Memory `Object2FloatMap<T>`

