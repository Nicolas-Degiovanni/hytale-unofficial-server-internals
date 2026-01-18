---
description: Architectural reference for Vector3dArrayCodec
---

# Vector3dArrayCodec

**Package:** com.hypixel.hytale.math.codec
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class Vector3dArrayCodec implements Codec<Vector3d> {
```

## Architecture & Concepts

The Vector3dArrayCodec is a specialized, low-level component within the engine's data serialization framework. Its sole responsibility is to translate the in-memory Vector3d object to and from a specific on-disk or network representation.

Architecturally, this codec implements a high-performance, compact serialization strategy by representing a Vector3d as a flat JSON or BSON array of three numbers, for example, `[1.0, 2.0, 3.0]`, rather than a more verbose object like `{ "x": 1.0, "y": 2.0, "z": 3.0 }`. This design choice prioritizes data density and parsing speed over human readability, which is a common trade-off in performance-critical game engine systems.

This class provides bidirectional translation for both the binary BSON format and the text-based JSON format. The `decodeJson` method is particularly notable for its use of a RawJsonReader, indicating a direct, token-level parsing implementation that avoids the overhead of intermediate object mapping libraries.

**WARNING:** This class is marked as **Deprecated**. It must not be used for new feature development. Its existence is for backward compatibility with legacy data formats. Engineering teams should identify and use the modern, non-deprecated codec for Vector3d serialization.

## Lifecycle & Ownership

-   **Creation:** Vector3dArrayCodec instances are not intended for manual creation. They are instantiated by a central `CodecRegistry` or a similar service provider during the engine's bootstrap sequence. The registry discovers and registers all available codecs to build a map of data types to their serializers.
-   **Scope:** This codec is a stateless singleton. A single instance is created and shared across the entire application for its full lifetime.
-   **Destruction:** The instance is destroyed and garbage collected only when the parent CodecRegistry is torn down, which typically occurs during a clean application shutdown.

## Internal State & Concurrency

-   **State:** **Stateless and Immutable.** This class contains no member fields and its methods operate exclusively on their input arguments. The output of any method is solely dependent on its inputs, with no side effects.
-   **Thread Safety:** **Fully thread-safe.** Due to its stateless nature, a single instance of Vector3dArrayCodec can be safely shared and invoked by multiple threads concurrently without any need for external locking or synchronization. This makes it suitable for use in high-throughput, multi-threaded data processing pipelines, such as asset loading or network packet processing.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Vector3d | O(1) | Deserializes a BSON array into a new Vector3d instance. Throws if the input is not a BsonArray of 3 numbers. |
| encode(Vector3d, ExtraInfo) | BsonValue | O(1) | Serializes a Vector3d instance into a new BsonArray containing its x, y, and z components. |
| decodeJson(RawJsonReader, ExtraInfo) | Vector3d | O(1) | Performs low-level, high-performance parsing of a JSON array from a character stream into a Vector3d. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema definition used for validation, defining the type as a fixed-size array of 3 numbers. |

## Integration Patterns

### Standard Usage

This codec is not designed to be used directly. It is an implementation detail of the higher-level serialization service. A developer would interact with a generic serialization interface, which internally dispatches to the correct codec based on the object's type.

```java
// CORRECT: Use a high-level serializer which finds this codec via a registry.
// The developer never references Vector3dArrayCodec directly.

Serializer serializer = context.getService(Serializer.class);
Vector3d myVector = new Vector3d(10, 20, 30);

// The serializer looks up the Codec<Vector3d> and invokes its encode method.
BsonValue bsonData = serializer.toBson(myVector);

// The serializer looks up the Codec<Vector3d> and invokes its decode method.
Vector3d deserializedVector = serializer.fromBson(bsonData, Vector3d.class);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance with `new Vector3dArrayCodec()`. The serialization system relies on a single, centrally registered instance to function correctly. Direct instantiation bypasses the registry and will fail in a production environment.
-   **Usage in New Systems:** Do not use this codec for any new data formats or systems. Its deprecated status indicates it has been superseded. Using it will introduce technical debt and potential forward-compatibility issues.
-   **Assuming Object Format:** Do not write tools or parsers that expect a Vector3d to be serialized as a JSON object. This codec strictly enforces the array format `[x, y, z]`.

## Data Pipeline

The Vector3dArrayCodec acts as a translation node in the data serialization and deserialization pipelines.

> **Serialization Flow:**
> In-Memory `Vector3d` Object -> **Vector3dArrayCodec.encode()** -> `BsonArray [x, y, z]` -> Network Buffer / File on Disk

> **Deserialization Flow:**
> Network Buffer / File on Disk -> BSON Parser -> `BsonArray [x, y, z]` -> **Vector3dArrayCodec.decode()** -> In-Memory `Vector3d` Object

