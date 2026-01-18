---
description: Architectural reference for LongArrayCodec
---

# LongArrayCodec

**Package:** com.hypixel.hytale.codec.codecs.array
**Type:** Utility

## Definition
```java
// Signature
public class LongArrayCodec implements Codec<long[]>, RawJsonCodec<long[]> {
```

## Architecture & Concepts
The LongArrayCodec is a specialized, low-level component within the engine's data serialization framework. It serves a singular, critical purpose: translating between the in-memory representation of a primitive long array (long[]) and its on-disk or over-the-wire BSON and JSON representations.

This class is a foundational building block, or *leaf codec*, in the composite structure of the serialization system. It does not handle complex objects. Instead, higher-level codecs, such as those for game entities or configuration objects, delegate the serialization of their long array fields to an instance of LongArrayCodec. This delegation is typically managed by a central CodecRegistry, which maps data types to their corresponding codecs.

Its dual implementation of both the BSON-centric Codec interface and the performance-oriented RawJsonCodec interface allows it to function efficiently in different serialization contexts without requiring an intermediate object representation.

## Lifecycle & Ownership
- **Creation:** Instantiated once during the application bootstrap phase by the core serialization system, typically a CodecRegistry or a similar factory. It is registered as the canonical handler for the long[] type.
- **Scope:** Singleton. A single, shared instance persists for the entire application lifetime. Its stateless nature permits this safely and efficiently.
- **Destruction:** The object is not explicitly destroyed. It is subject to garbage collection only upon final application shutdown when its ClassLoader is unloaded. No cleanup logic is required.

## Internal State & Concurrency
- **State:** This class is completely stateless. It holds no instance fields and does not cache data between calls. The static EMPTY_LONG_ARRAY constant is immutable. All operations are pure functions whose output depends solely on their inputs.
- **Thread Safety:** Inherently thread-safe. A single shared instance can be invoked concurrently from any number of threads without risk of race conditions or data corruption. No external locking or synchronization is necessary when using this codec.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | long[] | O(N) | Deserializes a BsonArray into a new primitive long array. Throws if the input is not a BsonArray or its elements are not 64-bit integers. |
| encode(long[], ExtraInfo) | BsonValue | O(N) | Serializes a primitive long array into a new BsonArray. |
| decodeJson(RawJsonReader, ExtraInfo) | long[] | O(N) | Deserializes a JSON array from a character stream. Implements an efficient, on-the-fly array growth strategy to minimize allocations. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a schema definition representing a long array, specifying it as an array of integers. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. It is used transparently by the main serialization service. The service identifies a field of type long[] and dispatches the serialization or deserialization task to the registered LongArrayCodec instance.

```java
// Hypothetical high-level object codec
// The Hytale engine performs this logic internally.
// You do not write this code.

// Encoding
long[] playerIds = { 1001L, 2050L, 4778L };
BsonValue bson = codecRegistry.getCodec(long[].class).encode(playerIds, ...);

// Decoding
long[] decodedIds = codecRegistry.getCodec(long[].class).decode(bson, ...);
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Avoid looking up and calling this codec directly. Let higher-level object codecs manage the serialization of their own fields. Manually encoding parts of an object breaks the atomicity of serialization.
- **Incorrect Type Handling:** Do not attempt to use this codec for an ArrayList of Long or any other collection type. It is strictly optimized for the primitive long[] array. Doing so will result in a ClassCastException at runtime.

## Data Pipeline
The LongArrayCodec acts as a translation node within a larger data pipeline.

> **Encoding Flow:**
> Game Object Field (long[]) -> Object Codec -> **LongArrayCodec** -> BsonArray -> BSON Document -> Network/Disk Stream

> **Decoding Flow:**
> Network/Disk Stream -> BSON Parser -> BsonArray -> Object Codec -> **LongArrayCodec** -> Game Object Field (long[])

