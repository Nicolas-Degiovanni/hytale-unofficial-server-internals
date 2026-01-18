---
description: Architectural reference for FloatArrayCodec
---

# FloatArrayCodec

**Package:** com.hypixel.hytale.codec.codecs.array
**Type:** Utility

## Definition
```java
// Signature
public class FloatArrayCodec implements Codec<float[]>, RawJsonCodec<float[]> {
```

## Architecture & Concepts
The FloatArrayCodec is a specialized, low-level component within the Hytale data serialization framework. Its sole responsibility is to provide a highly optimized translation layer between a primitive Java float array and its BSON or JSON representation.

This class acts as a concrete implementation of the generic Codec and RawJsonCodec interfaces. The engine's central CodecRegistry discovers and registers this codec to handle all serialization and deserialization requests for the `float[]` type. This design decouples game logic, which operates on native float arrays, from the underlying wire format used for networking or disk storage.

By implementing both interfaces, FloatArrayCodec ensures that float arrays can be processed from both the primary binary format (BSON) and secondary text-based formats (JSON), providing flexibility for different use cases like network replication, configuration files, or debugging tools.

### Lifecycle & Ownership
- **Creation:** Instantiated once by the core CodecRegistry during engine bootstrap. It is not intended for manual creation.
- **Scope:** Singleton-like. A single, shared instance persists for the entire application lifetime, serving all serialization requests for its target type.
- **Destruction:** The instance is eligible for garbage collection only upon application shutdown when the CodecRegistry is cleared.

## Internal State & Concurrency
- **State:** Stateless. This class contains no instance fields and maintains no internal state between calls. The `EMPTY_FLOAT_ARRAY` field is a static final constant used for optimization.
- **Thread Safety:** Inherently thread-safe. Due to its stateless nature, a single instance can be safely and concurrently invoked by multiple threads without requiring any external locking or synchronization mechanisms.

## API Surface
The public API provides the contract for the Codec system. These methods should not be invoked directly by game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | float[] | O(N) | Deserializes a BSON array into a new primitive float array. Throws if the input is not a BsonArray. |
| encode(float[], ExtraInfo) | BsonValue | O(N) | Serializes a primitive float array into a BsonArray. |
| decodeJson(RawJsonReader, ExtraInfo) | float[] | O(N) | Deserializes a JSON array from a raw character stream. This method is highly optimized for performance. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema definition describing this type as an array of numbers. |

*N = number of elements in the array.*

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. Instead, they interact with a higher-level serialization manager or a Codec-aware data structure, which uses the CodecRegistry to dispatch to the correct implementation.

```java
// The Codec system automatically finds and uses FloatArrayCodec
// when a float[] type is encountered.

// Example: A higher-level system serializing an object
GameObjectData data = new GameObjectData();
data.setVelocities(new float[]{10.5f, 0.0f, -5.2f});

// The serializer internally looks up the codec for float[]
BsonValue bson = Serializer.toBson(data);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FloatArrayCodec()`. The serialization framework depends on the single, globally registered instance to function correctly. Direct instantiation bypasses the registry and can lead to unpredictable behavior.
- **Manual Invocation:** Avoid calling `decode` or `encode` methods directly. The top-level serialization service provides necessary context and error handling that is bypassed with direct calls.

## Data Pipeline
FloatArrayCodec is a critical link in the data serialization and deserialization pipeline.

**Decoding (Network/Disk to Game Engine)**
> Flow:
> BSON BsonArray -> CodecRegistry -> **FloatArrayCodec.decode** -> primitive float[] -> Game State

**Encoding (Game Engine to Network/Disk)**
> Flow:
> Game State primitive float[] -> CodecRegistry -> **FloatArrayCodec.encode** -> BSON BsonArray -> Output Stream

