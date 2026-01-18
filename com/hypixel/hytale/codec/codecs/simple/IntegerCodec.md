---
description: Architectural reference for IntegerCodec
---

# IntegerCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class IntegerCodec implements Codec<Integer>, RawJsonCodec<Integer>, PrimitiveCodec {
```

## Architecture & Concepts
The IntegerCodec is a fundamental component of the Hytale serialization framework. It serves as a low-level, high-performance translator for the primitive Java **Integer** type. Its primary responsibility is to handle the bidirectional conversion between in-memory integer representations and their on-disk or network counterparts in both BSON and JSON formats.

As a **PrimitiveCodec**, it is treated as a foundational building block by the wider codec system. Higher-level, composite codecs (for example, a codec for a PlayerStats object) will delegate the serialization of their integer fields to this specialized implementation.

This class is not merely a data converter; it also integrates with the schema system via its **toSchema** methods. This allows the engine to define, validate, and reflect upon data structures that contain integer fields, ensuring data integrity across different game versions and systems.

## Lifecycle & Ownership
- **Creation:** A single, shared instance of IntegerCodec is instantiated by the central **CodecRegistry** during engine bootstrap. It is not intended for manual creation by developers.
- **Scope:** The instance is global and stateless. It persists for the entire lifetime of the application.
- **Destruction:** The object is de-referenced and eligible for garbage collection only when the main CodecRegistry is torn down, typically during a full client or server shutdown.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. The IntegerCodec contains no internal fields or member variables. All operations are pure functions, where the output depends solely on the input arguments.
- **Thread Safety:** **Fully Thread-Safe**. Due to its stateless design, a single instance can be safely shared and invoked by multiple threads simultaneously without any risk of race conditions or data corruption. This is critical for performance in multi-threaded environments, such as parallel chunk loading or network packet processing.

## API Surface
The public API provides the core serialization, deserialization, and schema generation contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Integer | O(1) | Deserializes a BSON value into a Java Integer. Throws **IllegalArgumentException** if the BSON number is a decimal. |
| encode(t, extraInfo) | BsonValue | O(1) | Serializes a Java Integer into a BSON BsonInt32 object. |
| decodeJson(reader, extraInfo) | Integer | O(1) | Deserializes an integer from a raw JSON stream reader. |
| toSchema(context) | Schema | O(1) | Generates a standard data schema definition for an integer type. |
| toSchema(context, def) | Schema | O(1) | Generates a data schema definition for an integer type, optionally including a default value. |

## Integration Patterns

### Standard Usage
This codec is almost never invoked directly. It is implicitly used by the serialization engine when it encounters an Integer field. The correct way to interact with it, if necessary, is by retrieving the shared instance from the central registry.

```java
// Retrieve the globally shared codec instance from the registry.
Codec<Integer> intCodec = CodecRegistry.getCodec(Integer.class);

// Example: Manually encode an integer for a special case or debugging.
BsonValue bsonRepresentation = intCodec.encode(1337, ExtraInfo.NONE);

// Example: Manually decode a BsonValue.
int decodedValue = intCodec.decode(new BsonInt32(1337), ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new IntegerCodec()**. This creates a redundant, unmanaged object that bypasses the engine's optimized singleton pattern. Always retrieve the shared instance from the CodecRegistry.
- **Type Mismatch:** Do not attempt to pass a BSON value representing a floating-point number (for example, BsonDouble) to the **decode** method. The codec contains a strict check and will throw an exception, as this indicates a data schema violation.

## Data Pipeline
The IntegerCodec is a specific step in a larger data serialization or deserialization pipeline. It does not manage I/O itself but operates on data streams provided by other systems.

> **Deserialization Flow (BSON):**
> Network Packet -> BsonReader -> **IntegerCodec.decode** -> Java Integer Object -> Game Logic

> **Serialization Flow (BSON):**
> Game Logic -> Java Integer Object -> **IntegerCodec.encode** -> BsonInt32 -> BsonWriter -> Network Packet

