---
description: Architectural reference for Vector2dArrayCodec
---

# Vector2dArrayCodec

**Package:** com.hypixel.hytale.math.codec
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class Vector2dArrayCodec implements Codec<Vector2d> {
```

## Architecture & Concepts

The Vector2dArrayCodec is a specialized component within the Hytale serialization framework responsible for translating a Vector2d object into a data format suitable for network transmission or disk storage, and vice-versa. It implements the generic Codec interface, marking it as a discoverable "translator" for the Vector2d type within the engine's central CodecRegistry.

Architecturally, this codec adopts a compact, array-based serialization strategy. It represents a Vector2d not as a self-describing object like `{"x": 1.0, "y": 2.0}`, but as a minimal two-element array of doubles: `[1.0, 2.0]`. This approach prioritizes data size and performance over readability and forward-compatibility.

**WARNING:** This class is annotated as **Deprecated**. This is a critical architectural signal that this implementation is considered obsolete. New systems should not use this codec. It is maintained for backward compatibility with older data formats. A more robust, object-based codec is the presumed modern replacement.

## Lifecycle & Ownership

-   **Creation:** Instances of this codec are not intended for manual creation. They are instantiated by a central service, likely a CodecRegistry or a dependency injection framework, during the application's bootstrap phase.
-   **Scope:** As a stateless utility, a single instance is shared globally and persists for the entire application lifetime.
-   **Destruction:** The instance is destroyed and garbage collected during application shutdown when the parent CodecRegistry is cleared.

## Internal State & Concurrency

-   **State:** This class is **stateless**. It contains no member variables and does not store or cache any data between calls. Each method call operates exclusively on the arguments provided.
-   **Thread Safety:** The component is inherently **thread-safe**. Due to its stateless nature, multiple threads can invoke the encode and decode methods concurrently without risk of race conditions or data corruption. No synchronization primitives are required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Vector2d | O(1) | Deserializes a BsonArray into a new Vector2d instance. Throws if the BsonValue is not an array with at least two numeric elements. |
| encode(Vector2d, ExtraInfo) | BsonValue | O(1) | Serializes a Vector2d instance into a two-element BsonArray. |
| decodeJson(RawJsonReader, ExtraInfo) | Vector2d | O(1) | Deserializes a JSON array from a raw character stream into a new Vector2d instance. Manually parses for high performance. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a data schema definition describing the format as a fixed-size array of two numbers. Used for data validation and tooling. |

## Integration Patterns

### Standard Usage

A developer should **never** interact with this class directly. The codec system is an abstraction layer. The correct pattern is to use a high-level serialization manager, which internally selects and delegates to the appropriate registered codec.

```java
// CORRECT: Use a higher-level serializer which finds this codec
// (Hypothetical example)
Serializer aSerializer = context.getService(Serializer.class);
Vector2d myVector = new Vector2d(10, 20);

// The serializer internally finds and uses Vector2dArrayCodec
BsonValue bson = aSerializer.toBson(myVector);
```

### Anti-Patterns (Do NOT do this)

-   **Usage in New Code:** The most severe anti-pattern is using this class for any new feature. Its Deprecated status explicitly forbids this. It exists only to read old data.
-   **Direct Instantiation:** Do not call `new Vector2dArrayCodec()`. This bypasses the engine's codec registry and can lead to inconsistent serialization behavior if a different codec for Vector2d is registered.
-   **Manual Invocation:** Avoid calling the `encode` or `decode` methods directly. This creates a hard dependency on a specific serialization format, making future data migrations difficult.

## Data Pipeline

The Vector2dArrayCodec acts as a transformation step in a larger data serialization or deserialization pipeline.

> **Serialization Flow:**
> Game Logic `Vector2d` Object -> Serializer Service -> **Vector2dArrayCodec.encode** -> `BsonArray` -> Network Buffer / File

> **Deserialization Flow:**
> Network Buffer / File -> BSON Parser -> `BsonArray` -> Serializer Service -> **Vector2dArrayCodec.decode** -> Game Logic `Vector2d` Object

