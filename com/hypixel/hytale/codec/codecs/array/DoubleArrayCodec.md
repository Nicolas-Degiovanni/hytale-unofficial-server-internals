---
description: Architectural reference for DoubleArrayCodec
---

# DoubleArrayCodec

**Package:** com.hypixel.hytale.codec.codecs.array
**Type:** Utility

## Definition
```java
// Signature
public class DoubleArrayCodec implements Codec<double[]>, RawJsonCodec<double[]> {
```

## Architecture & Concepts
The DoubleArrayCodec is a specialized, low-level component within the Hytale Codec Framework. Its sole responsibility is the bidirectional translation between the in-memory Java primitive `double[]` array and its serialized representations in both BSON and JSON formats.

This class acts as a fundamental data transformer. It is not intended for direct use by feature developers but is instead registered with and invoked by a higher-level serialization service, such as a central CodecRegistry or BsonSerializer. The framework identifies the need to process a `double[]` and dispatches the task to this specific codec. This design adheres to the single-responsibility principle, ensuring that the logic for handling this specific data type is encapsulated and optimized in one location.

By implementing both the Codec and RawJsonCodec interfaces, it signals its capability to handle the standard object-based BSON format and a performance-oriented raw JSON stream format.

## Lifecycle & Ownership
- **Creation:** An instance of DoubleArrayCodec is created during the application's bootstrap sequence by the core codec initialization service. It is immediately registered into a central codec lookup map or registry.
- **Scope:** This class is designed as a stateless singleton. A single instance persists for the entire lifetime of the application.
- **Destruction:** The instance is not explicitly destroyed. It is eligible for garbage collection only when the application is shutting down and its ClassLoader is unloaded. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The DoubleArrayCodec is **stateless**. It contains no instance fields and does not cache data between calls. All operations are pure functions, with output depending exclusively on the input arguments. The static `EMPTY_DOUBLE_ARRAY` field is a constant and does not constitute mutable state.
- **Thread Safety:** This class is inherently **thread-safe**. As a stateless utility, a single shared instance can be safely invoked by multiple threads concurrently without any risk of race conditions or data corruption. No external locking or synchronization is necessary when using this codec.

## API Surface
The public API provides the core serialization and deserialization contracts required by the Hytale Codec Framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | double[] | O(N) | Deserializes a BSON array into a primitive double array. N is the number of elements. |
| encode(double[], ExtraInfo) | BsonValue | O(N) | Serializes a primitive double array into a BSON array. N is the number of elements. |
| decodeJson(RawJsonReader, ExtraInfo) | double[] | O(N) | Deserializes a raw JSON stream into a primitive double array. N is the number of elements. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a schema object describing a numeric array for data validation and tooling. |

## Integration Patterns

### Standard Usage
Developers should **never** interact with this codec directly. Instead, they should rely on high-level serialization services which will delegate to this codec internally when appropriate.

```java
// The CodecRegistry or a Serializer service dispatches to DoubleArrayCodec
// internally based on the object's type.

// Example: Using a high-level serializer
BsonSerializer serializer = context.getService(BsonSerializer.class);
double[] physicsVector = new double[] { 9.81, 0.0, -1.5 };

// ENCODING: The serializer looks up the codec for double[] and invokes it.
BsonValue bsonRepresentation = serializer.encode(physicsVector);

// DECODING: The serializer is told the target type, finds the correct
// codec, and invokes it.
double[] decodedVector = serializer.decode(bsonRepresentation, double[].class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new DoubleArrayCodec()`. The codec framework relies on a single, centrally registered instance to function correctly. Bypassing the registry can lead to unpredictable serialization behavior.
- **Stateful Extension:** Do not extend this class to add state. The entire codec framework is architected on the assumption that codecs are stateless and thread-safe. Introducing state will break concurrency guarantees.
- **Incorrect BSON Input:** Passing a BsonValue that is not a BsonArray to the `decode` method will result in a runtime ClassCastException. The calling system is responsible for ensuring type correctness.

## Data Pipeline
The DoubleArrayCodec is a critical link in the data serialization and deserialization pipeline, converting between engine-native types and network-ready formats.

> **Encoding Flow:**
> In-Memory `double[]` -> `BsonSerializer.encode()` -> Codec Registry Lookup -> **DoubleArrayCodec.encode()** -> `BsonArray` -> Network/Disk Writer

> **Decoding Flow:**
> Network/Disk Reader -> `BsonArray` -> `BsonSerializer.decode()` -> Codec Registry Lookup -> **DoubleArrayCodec.decode()** -> In-Memory `double[]`

