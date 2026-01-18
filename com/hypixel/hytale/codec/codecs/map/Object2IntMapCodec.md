---
description: Architectural reference for Object2IntMapCodec
---

# Object2IntMapCodec<T>

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Transient Component

## Definition
```java
// Signature
public class Object2IntMapCodec<T> implements Codec<Object2IntMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Object2IntMapCodec is a specialized, high-performance component within the Hytale serialization framework. Its primary function is to translate between the in-memory representation of a fastutil Object2IntMap and its serialized form in either BSON or JSON.

This codec embodies the **Composition over Inheritance** principle. It does not handle the serialization of its map keys directly. Instead, it acts as a wrapper, delegating the key encoding and decoding logic to a separate, user-provided *child codec*. This design makes the Object2IntMapCodec extremely flexible and reusable. For example, the same codec can be configured to handle a map of String to integer or a map of a complex custom type to integer, simply by providing a different child codec at construction time.

Its direct integration with the fastutil library is a deliberate performance optimization, allowing the system to avoid the overhead of Java's standard boxed Integer objects in favor of primitive integers.

## Lifecycle & Ownership
- **Creation:** Instances are created manually by a developer or a higher-level factory system. Construction requires providing a child codec for the key type and a supplier function for creating new map instances. These instances are typically registered with a central Codec registry during application bootstrap.
- **Scope:** An Object2IntMapCodec instance is stateless and designed for reuse. It typically persists for the entire application session, held as a reference within a parent Codec registry.
- **Destruction:** The object has no explicit destruction logic. It is eligible for garbage collection when the Codec registry that holds it is unloaded, usually upon application termination.

## Internal State & Concurrency
- **State:** The Object2IntMapCodec is **immutable**. Its internal fields, including the child key codec and the map supplier, are final and set only during construction. It holds no mutable state related to any specific encoding or decoding operation.
- **Thread Safety:** This class is **fully thread-safe**. Due to its immutable nature, a single instance can be safely shared across multiple threads to perform concurrent serialization and deserialization operations without requiring any external locking or synchronization.

## API Surface
The public API is focused on the core responsibilities defined by the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Object2IntMap<T> | O(N) | Deserializes a BSON document into a new map instance. N is the number of key-value pairs. |
| encode(Object2IntMap<T>, ExtraInfo) | BsonValue | O(N) | Serializes a map into a BSON document. N is the number of key-value pairs. |
| decodeJson(RawJsonReader, ExtraInfo) | Object2IntMap<T> | O(N) | Deserializes a JSON object from a raw stream into a new map instance. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema representing a map with string-based keys and integer values. |
| getChildCodec() | Codec<T> | O(1) | Returns the wrapped codec responsible for handling the map's key type. |

## Integration Patterns

### Standard Usage
The codec is designed to be composed. A developer constructs an instance by providing the codec for the key and a factory for the desired map implementation.

```java
// Example: Creating a codec for a map of String to int
// using an Object2IntOpenHashMap for storage.
Codec<Object2IntMap<String>> stringToIntMapCodec = new Object2IntMapCodec<>(
    CODECS.STRING, // The child codec for the key type
    Object2IntOpenHashMap::new, // A supplier for the map implementation
    false // The resulting map will be mutable
);

// The codec can now be used to serialize and deserialize data.
BsonValue bson = stringToIntMapCodec.encode(myMap, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
- **Invalid Key Codec:** The child codec provided for the key type *must* serialize to a BsonString. BSON and JSON object keys are fundamentally strings. Providing a codec that serializes to a number or a nested object will result in a runtime ClassCastException.
- **Reusing Supplier Instances:** The map supplier function must create a **new, empty** map on every invocation. Returning a shared or pre-populated map instance from the supplier will lead to data corruption and unpredictable behavior across deserialization operations.

## Data Pipeline
The Object2IntMapCodec acts as a transformation stage in the broader data serialization pipeline.

> **Encoding Flow:**
> In-Memory `Object2IntMap<T>` -> **Object2IntMapCodec** -> `BsonDocument` -> Network or Disk Layer

> **Decoding Flow:**
> Network or Disk Layer -> `BsonDocument` -> **Object2IntMapCodec** -> In-Memory `Object2IntMap<T>` -> Game Logic

