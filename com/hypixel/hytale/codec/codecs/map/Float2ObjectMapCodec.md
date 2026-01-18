---
description: Architectural reference for Float2ObjectMapCodec
---

# Float2ObjectMapCodec

**Package:** com.hypixel.hytale.codec.codecs.map
**Type:** Utility

## Definition
```java
// Signature
public class Float2ObjectMapCodec<T> implements Codec<Float2ObjectMap<T>>, WrappedCodec<T> {
```

## Architecture & Concepts
The Float2ObjectMapCodec is a specialized component within the Hytale serialization framework responsible for translating between in-memory `fastutil` Float2ObjectMap instances and their BSON or JSON representations.

Its core design is based on the **Composition** pattern. It is a *composite codec* that does not manage the serialization of its value type T directly. Instead, it delegates this responsibility to a child Codec supplied during its construction. This decouples the logic for handling the map structure from the logic for handling the map's values, promoting modularity and reuse.

A key feature is the use of a Supplier to instantiate the map during decoding. This is a form of dependency injection that allows the system to provide different underlying map implementations (e.g., `Float2ObjectOpenHashMap`, `Float2ObjectArrayMap`) without altering the codec's logic, enabling performance tuning based on expected data characteristics.

The codec enforces a strict data contract: in the serialized form (BSON or JSON), the map keys must be strings that can be parsed as floating-point numbers. Failure to adhere to this contract will result in a CodecException during decoding.

## Lifecycle & Ownership
- **Creation:** Instances are created manually during the bootstrap phase of the codec system. A developer defines a new Float2ObjectMapCodec when a data model requires a map with float keys, composing it with the appropriate codec for the value type. It is not managed by a dependency injection container.
- **Scope:** An instance is typically created once and reused for the entire application lifetime. It is a stateless, reusable component whose lifecycle is bound to the Codec registry or the parent Codec that holds a reference to it.
- **Destruction:** The object is reclaimed by the Java garbage collector when it is no longer referenced. It holds no native resources and has no explicit destruction or `close` method.

## Internal State & Concurrency
- **State:** The Float2ObjectMapCodec is **immutable**. Its internal fields, including the child `valueCodec` and the map `supplier`, are set at construction and never modified. It does not cache any data between operations.
- **Thread Safety:** This class is **thread-safe** and fully re-entrant. The `encode` and `decode` methods operate only on their arguments and do not modify any internal state. The same instance can be safely shared across multiple threads to perform concurrent serialization and deserialization tasks.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Float2ObjectMap | O(N) | Deserializes a BsonDocument into a map. Throws CodecException on malformed data or key parsing failure. |
| encode(map, extraInfo) | BsonValue | O(N) | Serializes a map into a BsonDocument. Keys are converted to their string representation. |
| decodeJson(reader, extraInfo) | Float2ObjectMap | O(N) | Deserializes a JSON object from a stream into a map. Highly efficient for large payloads. |
| getChildCodec() | Codec | O(1) | Returns the composed codec used for serializing map values. |
| toSchema(context) | Schema | O(1) | Generates a formal schema definition for validation, requiring string keys that match a float pattern. |

## Integration Patterns

### Standard Usage
This codec is almost never used in isolation. It is intended to be composed into a larger codec that defines a complex data object.

```java
// In a static Codec definition class
// Defines a codec for an object that has a map of float->string properties

public static final Codec<MyConfig> CODEC = ObjectCodec.create(
    MyConfig::new,
    ObjectCodec.field(
        "properties",
        MyConfig::getProperties,
        new Float2ObjectMapCodec<>(
            Codec.STRING, // The child codec for the map's values
            Float2ObjectOpenHashMap::new, // The map implementation to use
            true // Make the decoded map unmodifiable
        )
    )
);
```

### Anti-Patterns (Do NOT do this)
- **Per-Operation Instantiation:** Do not create a `new Float2ObjectMapCodec()` for every object you serialize. It is a reusable component. Instantiate it once and store it in a static final field.
- **Invalid Key Types:** Do not attempt to serialize data where the source JSON or BSON contains non-numeric keys (e.g., `"name": "value"`). The decoder strictly expects keys like `"1.0"`, `"-50.25"`, etc., and will fail otherwise.
- **Modifying Unmodifiable Maps:** If the codec is constructed with `unmodifiable = true`, the returned map from a `decode` operation will throw an UnsupportedOperationException if modified. Treat the decoded object as a read-only data transfer object.

## Data Pipeline

The data flow for a decoding operation is a multi-stage process involving delegation to the child codec.

> Flow:
> BsonDocument -> **Float2ObjectMapCodec**.decode -> Map Instantiation (via Supplier) -> Iterate BSON entries -> Parse String Key to Float -> Delegate BsonValue to child **valueCodec** -> Populate Map -> Return Unmodifiable Map Wrapper

