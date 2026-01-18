---
description: Architectural reference for ByteCodec
---

# ByteCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility / Singleton

## Definition
```java
// Signature
public class ByteCodec implements Codec<Byte>, RawJsonCodec<Byte>, PrimitiveCodec {
```

## Architecture & Concepts
The ByteCodec is a foundational component within the Hytale serialization framework. Its primary responsibility is to provide a bidirectional, type-safe translation layer between the Java Byte primitive wrapper and its corresponding BSON and JSON representations.

As a *primitive* codec, it serves as a terminal node in the serialization graph. When a higher-level object serializer encounters a field of type Byte, it delegates the encoding or decoding task to this specialized class. The framework identifies this as a primitive handler via its implementation of the PrimitiveCodec marker interface.

A key architectural decision is the use of BsonInt32 as the wire format for a Java Byte. The BSON specification lacks a dedicated single-byte integer type. This implementation ensures compatibility and simplicity by using a standard 32-bit integer, while enforcing strict range validation during deserialization. This validation is critical for data integrity, preventing silent data truncation or corruption when a source provides an integer value outside the valid -128 to 127 range.

## Lifecycle & Ownership
- **Creation:** The ByteCodec is instantiated once by the central CodecRegistry during the engine's bootstrap sequence. It is then registered as the canonical handler for the java.lang.Byte class.
- **Scope:** This is a stateless singleton. A single, shared instance persists for the entire application lifetime. It is designed to be reused across all serialization and deserialization operations.
- **Destruction:** The instance is de-referenced and becomes eligible for garbage collection only when the parent CodecRegistry is destroyed, an event that typically coincides with application shutdown.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. The ByteCodec class contains no member fields. Its methods operate exclusively on the arguments provided, producing a result without any side effects or changes to internal state.
- **Thread Safety:** **Fully thread-safe**. Due to its stateless nature, a single ByteCodec instance can be safely and concurrently invoked by multiple threads without any need for external locking or synchronization. This is essential for high-performance, parallelized systems such as network packet processing and multi-threaded asset loading.

## API Surface
The public API provides the core serialization, deserialization, and schema-generation contracts required by the parent framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Byte | O(1) | Deserializes a BSON number into a Java Byte. Throws IllegalArgumentException if the value is outside the valid byte range. |
| encode(Byte, ExtraInfo) | BsonValue | O(1) | Serializes a Java Byte into a BSON BsonInt32. |
| decodeJson(RawJsonReader, ExtraInfo) | Byte | O(1) | Deserializes a JSON number from a stream into a Java Byte. Throws IllegalArgumentException for out-of-range values. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a corresponding IntegerSchema for config validation and tooling. |

## Integration Patterns

### Standard Usage
Direct interaction with the ByteCodec is rare and discouraged. The serialization framework uses it implicitly based on object field types. A developer relies on a higher-level serializer, which in turn dispatches to the registered ByteCodec.

```java
// The framework's Serializer is the intended entry point.
// It will look up and delegate to ByteCodec automatically
// for any fields of type Byte or byte.

// Example object with a byte field
public class PlayerState {
    byte health;
}

PlayerState state = new PlayerState();
state.health = 100;

// The serializer will find the 'health' field, query the CodecRegistry
// for the 'byte' type, and invoke ByteCodec.encode().
BsonValue bson = serializer.encode(state);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ByteCodec()`. The framework relies on a single, globally registered instance within the CodecRegistry. Bypassing the registry will lead to an unused object and potential framework errors.
- **Overriding Core Types:** Do not attempt to register a different codec for the Byte class. The engine expects the specific behavior and validation logic of this implementation. Overriding it can lead to subtle data corruption bugs and compatibility issues between client and server.

## Data Pipeline
The ByteCodec acts as a translation step in the middle of a larger data flow.

**Serialization (Encode) Flow:**
> Java Object (with `Byte` field) -> Serializer -> CodecRegistry -> **ByteCodec.encode()** -> BsonInt32 -> BSON Document -> Network or Disk Stream

**Deserialization (Decode) Flow:**
> Network or Disk Stream -> BSON Parser -> BsonInt32 -> Deserializer -> CodecRegistry -> **ByteCodec.decode()** -> Java Object (with `Byte` field)

