---
description: Architectural reference for StringCodec
---

# StringCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class StringCodec implements Codec<String>, RawJsonCodec<String> {
```

## Architecture & Concepts
The StringCodec is a fundamental, primitive component of the Hytale data serialization framework. It serves as a bidirectional translator between the engine's in-memory Java String representation and two primary on-disk or network data formats: BSON and JSON.

This class is not intended for direct use in game logic. Instead, it is a foundational building block used by more complex, composite codecs, such as an ObjectCodec. When a higher-level codec encounters a field of type String during serialization or deserialization, it delegates the conversion task to an instance of StringCodec. Its existence allows the rest of the serialization system to remain agnostic about the specific binary or text representation of a string.

It implements both the standard Codec interface for BSON operations and the RawJsonCodec interface for efficient, stream-based JSON processing.

### Lifecycle & Ownership
- **Creation:** A single, shared instance of StringCodec is instantiated by the core CodecRegistry during engine bootstrap. It is a default, built-in codec that is available immediately upon system initialization.
- **Scope:** The instance is a global singleton. It persists for the entire lifetime of the application.
- **Destruction:** The object requires no explicit cleanup and is garbage collected upon JVM shutdown.

## Internal State & Concurrency
- **State:** The StringCodec is **completely stateless and immutable**. It contains no member fields and its behavior is purely functional, determined exclusively by the arguments passed to its methods.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its stateless nature, a single instance can be safely shared and invoked by any number of threads concurrently without risk of race conditions or data corruption. No external locking is required.

## API Surface
The public contract is focused on the direct conversion of data and the generation of schema information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | String | O(1) | Deserializes a BsonValue (expected to be a BsonString) into a Java String. |
| encode(t, extraInfo) | BsonValue | O(1) | Serializes a Java String into a BsonString, which is a subtype of BsonValue. |
| decodeJson(reader, extraInfo) | String | O(N) | Deserializes a string from a JSON stream. Complexity is linear to string length. |
| toSchema(context) | Schema | O(1) | Generates a standard StringSchema definition for this type. |
| toSchema(context, def) | Schema | O(1) | Generates a StringSchema definition that includes an optional default value. |

## Integration Patterns

### Standard Usage
Developers will almost never interact with this class directly. Its usage is implicit, managed by the higher-level codec system. When you define a data-transfer object with a String field, the serialization framework automatically resolves and uses this codec.

The following example is conceptual, demonstrating how a registry might provide access to the codec.

```java
// In practice, the CodecRegistry is an internal system.
// This demonstrates the resolution and use of the shared instance.
Codec<String> stringCodec = CodecRegistry.getCodec(String.class);

// Encoding a string to BSON
BsonValue bsonData = stringCodec.encode("Hytale", ExtraInfo.NONE);

// Decoding a string from BSON
String originalString = stringCodec.decode(bsonData, ExtraInfo.NONE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StringCodec()`. This creates a redundant, unnecessary object. The serialization framework provides a globally shared instance that must be used.
- **Redundant Registration:** Do not attempt to register this codec for the String class in a custom CodecRegistry. The system relies on the default, built-in binding. Overriding it can lead to unpredictable serialization behavior.

## Data Pipeline
The StringCodec acts as a terminal node in a data flow, performing the final conversion step between a serialized format and a Java object.

> **Flow (Decoding):**
> BSON Payload / JSON Stream → Core Codec Framework → **StringCodec** → Java String Instance

> **Flow (Encoding):**
> Java String Instance → Core Codec Framework → **StringCodec** → BSON Payload / JSON Stream

