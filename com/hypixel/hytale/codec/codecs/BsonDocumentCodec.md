---
description: Architectural reference for BsonDocumentCodec
---

# BsonDocumentCodec

**Package:** com.hypixel.hytale.codec.codecs
**Type:** Utility

## Definition
```java
// Signature
@Deprecated
public class BsonDocumentCodec implements Codec<BsonDocument> {
```

## Architecture & Concepts
The BsonDocumentCodec is a foundational component within the Hytale serialization framework. It serves as a specialized **Codec** responsible for the direct serialization and deserialization of the BsonDocument type, which is a fundamental data structure for representing schemaless, document-oriented data.

Architecturally, this codec acts as a "pass-through" or identity transformer for BSON-to-BSON operations. Its primary role is not to perform complex data conversion but to handle the raw BsonDocument container itself. The most significant function is its ability to bridge text-based JSON data into the engine's binary BSON format via the decodeJson method.

**WARNING:** This class is marked as **Deprecated**. Its presence indicates a legacy implementation. New systems should avoid using this codec directly and instead rely on schema-driven serialization mechanisms that provide type safety and validation. This codec is likely maintained for backward compatibility with older data formats or configurations.

### Lifecycle & Ownership
- **Creation:** Instantiated by a central CodecRegistry or a similar service provider during the engine's bootstrap sequence. The framework discovers and registers all available Codec implementations.
- **Scope:** Singleton. A single instance is created and shared across the entire application. It persists for the duration of the client or server session.
- **Destruction:** The instance is eligible for garbage collection upon application shutdown when the central CodecRegistry is cleared.

## Internal State & Concurrency
- **State:** This class is completely stateless. It contains no member fields and all its methods operate exclusively on the arguments provided.
- **Thread Safety:** Inherently thread-safe. As a stateless utility, its methods can be invoked concurrently from any thread without risk of race conditions or side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | BsonDocument | O(1) | Casts a generic BsonValue to a BsonDocument. Throws a runtime exception if the value is not a document. |
| encode(BsonDocument, ExtraInfo) | BsonValue | O(1) | Returns the input BsonDocument without modification. Acts as an identity function. |
| decodeJson(RawJsonReader, ExtraInfo) | BsonDocument | O(N) | Parses a JSON data stream into a BsonDocument. Complexity is linear to the size of the input JSON. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a generic ObjectSchema, signifying that the data is an unstructured key-value map. |

## Integration Patterns

### Standard Usage
This codec is not intended for direct use by feature developers. It is invoked implicitly by the higher-level serialization engine when it encounters a field or object of type BsonDocument. The system retrieves the registered codec and delegates the operation to it.

```java
// System-level code (Illustrative)
// A serialization service would look up the codec and use it.
Codec<BsonDocument> codec = codecRegistry.getCodec(BsonDocument.class);
BsonDocument data = codec.decode(someBsonValue, extraInfo);
```

### Anti-Patterns (Do NOT do this)
- **Usage in New Systems:** The primary anti-pattern is using this class in any new development. Its deprecated status signals that a more modern, schema-aware alternative must be used.
- **Direct Instantiation:** Do not create instances using `new BsonDocumentCodec()`. The serialization framework is responsible for the lifecycle of all codecs. Direct instantiation bypasses the central registry and can lead to unpredictable behavior.

## Data Pipeline
The BsonDocumentCodec is a transformation step in several data pipelines, primarily for configuration loading and raw network packet handling.

> **JSON to BSON Flow:**
> JSON File/Stream -> RawJsonReader -> **BsonDocumentCodec**.decodeJson -> BsonDocument (In-Memory)

> **BSON Deserialization Flow:**
> Raw BsonValue -> **BsonDocumentCodec**.decode -> BsonDocument (In-Memory)

