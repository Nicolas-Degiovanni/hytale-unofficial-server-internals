---
description: Architectural reference for FloatRangeArrayCodec
---

# FloatRangeArrayCodec

**Package:** com.hypixel.hytale.math.codec
**Type:** Utility

## Definition
```java
// Signature
public class FloatRangeArrayCodec implements Codec<FloatRange>, ValidatableCodec<FloatRange> {
```

## Architecture & Concepts
The FloatRangeArrayCodec is a specialized component within Hytale's data serialization framework. Its primary function is to act as a bidirectional translator for the FloatRange data type, converting the in-memory Java object to and from its BSON and JSON representations.

This codec is fundamental to the engine's data-driven architecture. It enables designers and developers to define numeric ranges in external configuration files (e.g., for weapon damage, AI detection distance, or procedural generation parameters) and have them safely loaded into the game engine.

By implementing the ValidatableCodec interface, it enforces data integrity at the deserialization boundary. This prevents logically invalid data, such as a minimum value greater than the maximum, from propagating into the game state and causing unpredictable behavior. The inclusion of a `toSchema` method indicates integration with a schema-driven system, used for tooling, automated validation, and configuration file linting.

## Lifecycle & Ownership
- **Creation:** This class is not intended for manual instantiation. It is discovered and instantiated by the central CodecRegistry during the engine's bootstrap sequence. The registry is responsible for managing the single, canonical instance of this codec.
- **Scope:** The FloatRangeArrayCodec is a stateless object whose lifetime is tied to the CodecRegistry. It persists for the entire application session once initialized.
- **Destruction:** The object is eligible for garbage collection only when the CodecRegistry is torn down, typically during a full engine shutdown.

## Internal State & Concurrency
- **State:** The FloatRangeArrayCodec is **completely stateless**. It retains no data between method invocations. All operations are performed exclusively on the inputs provided in the method arguments. This design is critical for a reliable and reusable serialization component.
- **Thread Safety:** As a direct consequence of its stateless design, this class is inherently **thread-safe**. A single instance can be safely invoked by multiple threads concurrently without the need for external locking or synchronization.

## API Surface
The public API provides the core serialization, deserialization, and validation contract required by the Hytale codec framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | FloatRange | O(1) | Deserializes a BSON array of two numbers into a FloatRange object. |
| encode(FloatRange, ExtraInfo) | BsonValue | O(1) | Serializes a FloatRange object into a BSON array of two numbers. |
| decodeJson(RawJsonReader, ExtraInfo) | FloatRange | O(1) | Deserializes a JSON array from a raw stream into a FloatRange object. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a formal data schema describing the expected `[min, max]` array structure. |
| validate(FloatRange, ExtraInfo) | void | O(1) | Executes business logic validation. Fails if the range's minimum exceeds its maximum. |

## Integration Patterns

### Standard Usage
Developers do not interact with this codec directly. It is invoked transparently by higher-level systems like an asset loader or configuration manager via the central CodecRegistry. The registry selects this codec based on the target type being FloatRange.

```java
// Example of a system using the CodecRegistry, which in turn uses this codec.
CodecRegistry registry = context.getService(CodecRegistry.class);
BsonArray bsonData = new BsonArray(Arrays.asList(new BsonDouble(10.0), new BsonDouble(50.0)));

// The registry automatically finds and uses FloatRangeArrayCodec here.
FloatRange damageRange = registry.decode(bsonData, FloatRange.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new FloatRangeArrayCodec()`. The codec system relies on a single, registered instance to function correctly. Direct instantiation bypasses the registry and will lead to system failure.
- **Manual Validation Calls:** Do not call the `validate` method directly. Validation is an integral part of the deserialization pipeline and is managed automatically by the framework when decoding data.

## Data Pipeline
The FloatRangeArrayCodec sits at a critical juncture in the data flow between persistent storage and the live game engine.

**Deserialization (Loading) Flow:**
> BSON or JSON File -> Engine BSON/JSON Parser -> CodecRegistry -> **FloatRangeArrayCodec.decode** -> **FloatRangeArrayCodec.validate** -> In-Memory FloatRange Object

**Serialization (Saving) Flow:**
> In-Memory FloatRange Object -> CodecRegistry -> **FloatRangeArrayCodec.encode** -> BSON/JSON Serializer -> BSON or JSON File

