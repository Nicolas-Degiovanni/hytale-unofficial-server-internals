---
description: Architectural reference for LongCodec
---

# LongCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class LongCodec implements Codec<Long>, RawJsonCodec<Long>, PrimitiveCodec {
```

## Architecture & Concepts
The LongCodec is a foundational component within the Hytale serialization framework. It serves as a specialized, high-performance translator for the Java Long data type. Its primary responsibility is to convert 64-bit integer values between their in-memory Java representation and their on-disk or on-wire formats, specifically BSON and a raw JSON stream.

As a class that implements the PrimitiveCodec marker interface, it signals to the parent Codec system that Long is a terminal or "leaf" type in the object graph. The serialization engine will not attempt to recursively inspect the fields of a Long; instead, it delegates directly to this codec for immediate conversion. This design is critical for performance, preventing unnecessary reflection and type analysis on fundamental data types.

The LongCodec is a core building block, used implicitly by more complex codecs that serialize objects containing Long fields. Developers will rarely interact with this class directly, but its correct function is essential for the integrity of all data persistence and network communication.

### Lifecycle & Ownership
- **Creation:** Instances of LongCodec are intended to be created and managed by a central CodecRegistry or a similar service provider during application bootstrap. The system pre-registers this codec against the java.lang.Long class type.
- **Scope:** A single, shared instance typically persists for the entire application lifetime. Due to its stateless nature, there is no overhead or risk associated with reusing the same instance across the entire system.
- **Destruction:** The object has no explicit destruction logic. It is eligible for garbage collection when the application shuts down and its managing CodecRegistry is unloaded.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. The LongCodec class contains no member fields. Its methods operate exclusively on the arguments provided, producing a new output value without causing any side effects or modifying internal state.
- **Thread Safety:** **Fully thread-safe**. As a stateless utility, an instance of LongCodec can be safely invoked from multiple threads concurrently without any need for external synchronization or locks. This is a crucial characteristic for its use in multi-threaded networking and world-processing systems.

## API Surface
The public API provides the contract for encoding, decoding, and schema generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Long | O(1) | Converts a BSON numeric value to a Java Long. Throws IllegalArgumentException if the BSON value is a decimal. |
| encode(t, extraInfo) | BsonValue | O(1) | Converts a Java Long to a BSON BsonInt64 value. |
| decodeJson(reader, extraInfo) | Long | O(1) | Reads and parses a 64-bit integer from a raw JSON stream. |
| toSchema(context) | Schema | O(1) | Generates a generic IntegerSchema definition for this type. |
| toSchema(context, def) | Schema | O(1) | Generates an IntegerSchema definition with a specified default value. |

## Integration Patterns

### Standard Usage
A developer should never invoke LongCodec directly. Instead, interact with the primary serialization service, which will delegate to the registered LongCodec instance internally when it encounters a Long field.

```java
// Correct Usage: Rely on the parent Codec system
// Assume 'codecService' is the central serialization engine.

// Encoding an object containing a Long
PlayerStats stats = new PlayerStats();
stats.setPlayerId(123456789012345L); // This is a long
BsonDocument bsonData = codecService.toBson(stats);

// Decoding an object containing a Long
PlayerStats decodedStats = codecService.fromBson(bsonData, PlayerStats.class);
long playerId = decodedStats.getPlayerId();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call new LongCodec(). The codec system relies on a single, registered instance to guarantee consistent behavior. Manually creating instances bypasses the central registry and can lead to unpredictable serialization errors.
- **Mismatched Type Registration:** Never register LongCodec for a different data type (e.g., Integer or String). The codec performs specific checks and conversions that will fail if the input type is not a Long.

## Data Pipeline
The LongCodec acts as a direct translation node in the data serialization and deserialization pipelines.

**Decoding Flow (BSON to Java)**
> BSON BsonInt64 -> **LongCodec::decode** -> Java Long

**Encoding Flow (Java to BSON)**
> Java Long -> **LongCodec::encode** -> BSON BsonInt64

