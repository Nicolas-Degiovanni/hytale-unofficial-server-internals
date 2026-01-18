---
description: Architectural reference for ShortCodec
---

# ShortCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class ShortCodec implements Codec<Short>, RawJsonCodec<Short>, PrimitiveCodec {
```

## Architecture & Concepts
The ShortCodec is a foundational component within the Hytale serialization framework. Its sole responsibility is to act as a bidirectional translator for the Java Short primitive wrapper type, handling conversions between the in-memory representation and two serialized formats: BSON and JSON.

As a specialized implementation of the Codec interface, it is automatically discovered and managed by a central CodecRegistry. The framework delegates all serialization and deserialization operations for Short fields to this class.

The implementation of RawJsonCodec provides a distinct, optimized path for handling text-based configuration or data files, bypassing the BSON object model for direct stream processing. The PrimitiveCodec marker interface signals to the framework that this codec handles a fundamental data type, which may allow for certain optimizations or assumptions during schema generation and validation.

A critical design feature is its strict range validation. During deserialization from either BSON or JSON, it ensures the incoming integer value fits within the bounds of a Java Short (-32,768 to 32,767). This defensive programming prevents data corruption and overflow errors that could arise from malformed or malicious payloads.

### Lifecycle & Ownership
- **Creation:** Instantiated once at application startup by the core codec registration system. It is not intended for manual creation by developers.
- **Scope:** Singleton. A single, shared instance persists for the entire application lifecycle. Its stateless nature makes this model highly efficient.
- **Destruction:** The instance is eligible for garbage collection only upon application shutdown when the central CodecRegistry is cleared.

## Internal State & Concurrency
- **State:** The ShortCodec is **immutable and stateless**. It contains no member fields and its behavior is determined exclusively by the arguments passed to its methods.
- **Thread Safety:** This class is **inherently thread-safe**. Its methods are pure functions, allowing concurrent invocation from multiple threads without any risk of race conditions or need for external synchronization. This is essential for high-throughput networking and data processing systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Short | O(1) | Deserializes a BsonValue into a Short. Throws IllegalArgumentException if the value is out of range. |
| encode(Short, ExtraInfo) | BsonValue | O(1) | Serializes a Short into a BsonInt32. |
| decodeJson(RawJsonReader, ExtraInfo) | Short | O(1) | Deserializes a JSON number from a stream into a Short. Throws IllegalArgumentException if the value is out of range. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a schema representation for this type, returning an IntegerSchema. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with the ShortCodec directly. It is designed to be used implicitly by the higher-level serialization engine. The engine resolves the appropriate codec based on the type of a field and dispatches the operation accordingly.

```java
// FRAMEWORK-LEVEL USAGE (NOT for typical game developers)

// 1. The framework retrieves the registered codec for the Short type.
Codec<Short> codec = codecRegistry.getCodec(Short.class);

// 2. The framework invokes the codec to process a value.
BsonValue bsonRepresentation = codec.encode((short) 42, ...);
Short javaRepresentation = codec.decode(bsonRepresentation, ...);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ShortCodec()`. The serialization framework manages a shared, singleton instance. Creating your own bypasses the central registry and is wasteful.
- **Manual Invocation:** Directly calling `shortCodec.encode()` on a value is a code smell. You should be serializing a container object (e.g., a PlayerState) and allowing the framework to recursively handle its fields. Bypassing the framework can lead to incomplete or incorrect serialization.

## Data Pipeline
The ShortCodec is a specific node in the overall data serialization and deserialization pipeline.

> **Encoding Flow:**
> Java Object Field (Short) -> Serialization Engine -> **ShortCodec.encode** -> BsonInt32 -> BSON Document -> Network Buffer

> **BSON Decoding Flow:**
> Network Buffer -> BSON Parser -> BsonInt32 -> **ShortCodec.decode** -> Java Object Field (Short)

> **JSON Decoding Flow:**
> JSON File -> RawJsonReader Stream -> **ShortCodec.decodeJson** -> Java Object Field (Short)

