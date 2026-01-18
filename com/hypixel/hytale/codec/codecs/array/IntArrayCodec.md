---
description: Architectural reference for IntArrayCodec
---

# IntArrayCodec

**Package:** com.hypixel.hytale.codec.codecs.array
**Type:** Utility

## Definition
```java
// Signature
public class IntArrayCodec implements Codec<int[]>, RawJsonCodec<int[]> {
```

## Architecture & Concepts
The IntArrayCodec is a low-level, specialized component within the engine's data serialization framework. Its sole responsibility is the bidirectional conversion of a primitive Java integer array (int[]) to and from BSON and raw JSON representations.

This class acts as a concrete implementation of the generic Codec and RawJsonCodec interfaces. It is not intended for direct use by feature developers but is instead registered with and managed by a central CodecRegistry. The registry delegates serialization and deserialization tasks for any field or object of type int[] to this component.

The dual-interface implementation is critical for supporting two distinct use cases:
1.  **BSON Serialization (via Codec):** Used for efficient, binary-formatted data transfer over the network or for saving to disk, such as in player data or world chunks.
2.  **JSON Serialization (via RawJsonCodec):** Used for human-readable and editable data, primarily for configuration files, asset definitions, or debugging outputs.

## Lifecycle & Ownership
- **Creation:** A single instance of IntArrayCodec is instantiated at application startup by the core serialization system, typically a CodecRegistry or a similar service locator. It is designed to be a shared, global instance.
- **Scope:** The instance is application-scoped. It persists for the entire lifetime of the client or server process.
- **Destruction:** The object is garbage collected during application shutdown when the managing registry is destroyed. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The IntArrayCodec is **stateless**. It contains no instance fields and does not store or cache any data between calls. All operations are pure functions of their inputs.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads concurrently without the need for locks or any other synchronization primitives. This is essential for high-performance, multi-threaded environments like a game server.

## API Surface
The public API provides the core serialization and deserialization contracts.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | int[] | O(N) | Deserializes a BsonArray into a primitive int array. Throws a runtime exception if the input is not a BsonArray. |
| encode(int[], ExtraInfo) | BsonValue | O(N) | Serializes a primitive int array into a BsonArray. |
| decodeJson(RawJsonReader, ExtraInfo) | int[] | O(N) | Deserializes a JSON array from a stream into a primitive int array. Optimized for performance by minimizing reallocations. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a metadata schema describing the data format as an array of integers. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. The engine's serialization service abstracts away the specific codec implementation. The framework automatically selects IntArrayCodec when processing an int array.

A hypothetical interaction with the parent system would look like this:
```java
// The CodecRegistry would internally locate and use IntArrayCodec for this operation
int[] myData = {10, 20, 30};
BsonValue bsonData = codecRegistry.encode(myData);

// ... network transmission ...

int[] decodedData = codecRegistry.decode(bsonData, int[].class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IntArrayCodec()`. This is wasteful and bypasses the centralized registry, which may have extended or wrapped functionality. Always rely on the engine's provided serialization services.
- **Invalid Input Type:** Passing a BsonValue that is not a BsonArray to the decode method will result in a fatal BsonInvalidOperationException. The caller is responsible for ensuring type correctness before delegation.

## Data Pipeline
The IntArrayCodec is a specific step in a larger data serialization pipeline.

**BSON Pipeline (Network/Storage)**
> Flow:
> Game State (int[]) -> **IntArrayCodec.encode** -> BsonArray -> Network Buffer / File

> Flow:
> Network Buffer / File -> BsonArray Parser -> **IntArrayCodec.decode** -> Game State (int[])

**JSON Pipeline (Configuration)**
> Flow:
> Config File (.json) -> RawJsonReader -> **IntArrayCodec.decodeJson** -> Configuration Object (int[])

