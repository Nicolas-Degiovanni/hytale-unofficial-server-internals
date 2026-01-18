---
description: Architectural reference for BoolDoublePairCodec
---

# BoolDoublePairCodec

**Package:** com.hypixel.hytale.server.core.codec
**Type:** Utility / Singleton

## Definition
```java
// Signature
public class BoolDoublePairCodec implements Codec<BoolDoublePair> {
```

## Architecture & Concepts
The BoolDoublePairCodec is a specialized serialization component responsible for translating the in-memory BoolDoublePair data structure to and from its BSON representation. It is a fundamental part of the engine's data serialization pipeline, which ensures that complex game data can be efficiently stored, networked, and retrieved.

This codec implements a custom, compact serialization format. Instead of using a standard BSON document like *{ "flag": true, "value": 12.3 }*, it encodes the boolean and double into a single BSON value. This is achieved through two distinct strategies:

1.  **String-based Encoding:** A BsonString is used where a tilde prefix (~) indicates the boolean is *true*. The absence of the prefix indicates *false*. This format is optimized for human readability and configuration files.
2.  **Numeric-based Encoding:** A raw BsonDouble value is interpreted as a pair where the boolean is implicitly *false*. This provides a fallback for data originating from systems that are not aware of the custom string format.

This dual-path approach provides both flexibility and backward compatibility within the broader Hytale data ecosystem.

### Lifecycle & Ownership
-   **Creation:** This codec is designed to be instantiated by a central CodecRegistry or a similar service locator during application bootstrap. The framework is responsible for discovering and registering all Codec implementations.
-   **Scope:** Application-scoped. A single instance is registered and shared for the entire lifetime of the server or client process. It persists as long as the serialization system is active.
-   **Destruction:** The object is garbage collected when the application shuts down and its parent CodecRegistry is cleared. It requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** This class is **stateless**. It contains no instance fields and its only static field, PATTERN, is an immutable, pre-compiled regular expression. All operations are pure functions of their inputs.
-   **Thread Safety:** Inherently **thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads concurrently without any risk of race conditions or data corruption. This is a critical design feature for high-throughput server environments where network packets are processed in parallel.

## API Surface
The public contract is defined by the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | BoolDoublePair | O(N) | Deserializes a BsonValue into a BoolDoublePair. N is the length of the string if the input is a BsonString. Handles both string and numeric BSON types. |
| encode(BoolDoublePair, ExtraInfo) | BsonValue | O(N) | Serializes a BoolDoublePair into a BsonString. N is the number of digits in the double. The boolean value determines the presence of the tilde prefix. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a validation schema that accepts either a Number or a String matching the custom format. This is used by schema validation systems. |

## Integration Patterns

### Standard Usage
This codec is not intended for direct use by application-level code. It is invoked implicitly by a higher-level serialization facade, such as a BsonMapper or Serializer service. The framework automatically selects the correct codec based on the target field's type.

```java
// A developer interacts with a high-level mapper, not the codec itself.
// The mapper will discover and use BoolDoublePairCodec internally.

// Example object containing the target type
public class PlayerMovement {
    public BoolDoublePair relativeX;
    public BoolDoublePair relativeY;
}

// Correct usage via a central serialization service
BsonMapper mapper = context.getService(BsonMapper.class);
PlayerMovement movement = mapper.fromBson(bsonDocument, PlayerMovement.class);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BoolDoublePairCodec()`. The codec system relies on a single, registered instance. Manual creation bypasses the framework and can lead to unpredictable serialization behavior.
-   **Manual Invocation:** Avoid calling `codec.encode()` or `codec.decode()` directly. This creates a tight coupling to the BSON implementation details and violates the abstraction provided by the serialization framework. Always use the central mapper or serializer service.

## Data Pipeline
The BoolDoublePairCodec acts as a transformation step within the broader data serialization and deserialization pipelines.

**Deserialization (Network to Game State):**
> Flow:
> BSON Byte Stream -> BSON Parser -> BsonValue -> **BoolDoublePairCodec.decode** -> In-Memory BoolDoublePair -> Game Logic

**Serialization (Game State to Network):**
> Flow:
> Game Logic -> In-Memory BoolDoublePair -> **BoolDoublePairCodec.encode** -> BsonValue -> BSON Serializer -> BSON Byte Stream

