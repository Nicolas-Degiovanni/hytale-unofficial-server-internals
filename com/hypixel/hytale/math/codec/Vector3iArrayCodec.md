---
description: Architectural reference for Vector3iArrayCodec
---

# Vector3iArrayCodec

**Package:** com.hypixel.hytale.math.codec
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class Vector3iArrayCodec implements Codec<Vector3i> {
```

## Architecture & Concepts

The Vector3iArrayCodec is a specific, low-level component within the engine's data serialization framework. It implements the generic Codec interface, providing a concrete strategy for converting a Vector3i object to and from a serialized representation.

Architecturally, this codec represents a design choice favoring data compactness over readability. It serializes a Vector3i object, which has named fields (x, y, z), into a simple 3-element integer array. This approach minimizes the data footprint, which is advantageous for high-volume network traffic or storage, but sacrifices the self-describing nature of an object-based format like `{ "x": 10, "y": 20, "z": 30 }`.

This class supports two primary serialization formats:
1.  **BSON:** A binary-safe representation used for efficient network transport and storage.
2.  **JSON:** A human-readable text representation, handled by a manual, high-performance raw JSON parser.

**Warning:** This class is marked as **Deprecated**. Its usage is strongly discouraged in new systems. It likely exists for backward compatibility with older data formats. Newer implementations should use a more standard, object-based codec that provides better data clarity and forward compatibility.

## Lifecycle & Ownership

-   **Creation:** Instances of Vector3iArrayCodec are not intended to be created directly by application code. They are typically instantiated once by a central CodecRegistry or a similar service during the engine's bootstrap sequence. The registry scans, discovers, and maps this codec to the Vector3i class.
-   **Scope:** As a stateless utility, a single shared instance of this codec persists for the entire application session. It follows a Flyweight pattern, where one instance serves all serialization requests for Vector3i objects.
-   **Destruction:** The singleton instance is destroyed and garbage collected only when its parent CodecRegistry is torn down, which typically occurs during a graceful application shutdown.

## Internal State & Concurrency

-   **State:** This class is **stateless**. It contains no member fields and its behavior depends exclusively on the arguments passed to its methods. All operations are pure functions.
-   **Thread Safety:** The component is inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads concurrently without requiring any locks or synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Vector3i | O(1) | Deserializes a BsonArray into a new Vector3i instance. Throws if the input is not a BsonArray or has fewer than 3 elements. |
| encode(Vector3i, ExtraInfo) | BsonValue | O(1) | Serializes a Vector3i instance into a new BsonArray containing its x, y, and z components. |
| decodeJson(RawJsonReader, ExtraInfo) | Vector3i | O(1) | Deserializes a JSON array from a raw character stream into a new Vector3i instance. Performs no heap allocations beyond the final object. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema definition describing the serialized format as a fixed-size array of three numbers. |

## Integration Patterns

### Standard Usage

Developers should never invoke this codec directly. The serialization system abstracts away the specific codec implementation. A higher-level service uses a registry to find the appropriate codec for a given type at runtime.

```java
// CORRECT: The system finds and uses the codec transparently.
// A developer would typically interact with a higher-level manager.

// Hypothetical example of how the engine uses it internally:
CodecRegistry registry = context.getService(CodecRegistry.class);
Codec<Vector3i> vectorCodec = registry.getCodec(Vector3i.class);

// Serialization
Vector3i myVector = new Vector3i(10, 20, 30);
BsonValue bsonData = vectorCodec.encode(myVector, ExtraInfo.NONE);

// Deserialization
Vector3i decodedVector = vectorCodec.decode(bsonData, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new Vector3iArrayCodec()`. The codec system relies on a single, managed instance within the CodecRegistry to function correctly. Direct instantiation bypasses this system.
-   **Usage in New Code:** The most critical anti-pattern is using this codec for any new feature or data format. The Deprecated annotation serves as a strict warning that it is obsolete and may be removed in a future version. Relying on it introduces technical debt and risks breaking compatibility.

## Data Pipeline

The Vector3iArrayCodec acts as a translation node in the data serialization and deserialization pipelines.

> **Serialization Flow:**
> Game Object (Vector3i) -> Serialization Service -> CodecRegistry lookup -> **Vector3iArrayCodec.encode()** -> BsonArray -> Network Buffer / File

> **Deserialization Flow:**
> Network Buffer / File -> BSON Parser -> BsonArray -> Deserialization Service -> CodecRegistry lookup -> **Vector3iArrayCodec.decode()** -> Game Object (Vector3i)

