---
description: Architectural reference for BooleanCodec
---

# BooleanCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class BooleanCodec implements Codec<Boolean>, RawJsonCodec<Boolean>, PrimitiveCodec {
```

## Architecture & Concepts
The BooleanCodec is a foundational component within the Hytale serialization framework. Its sole responsibility is to provide a bidirectional translation layer between the in-memory Java Boolean type and its serialized representations in both BSON (binary) and JSON (text) formats.

As a *simple* codec, it handles a single, indivisible data type. It acts as a fundamental building block for the serialization of complex objects. When the framework encounters a boolean field in a larger data structure, it delegates the encoding or decoding of that specific field to this specialized class.

The implementation of three distinct interfaces signals its versatile role:
*   **Codec<Boolean>:** Defines the contract for BSON serialization and deserialization.
*   **RawJsonCodec<Boolean>:** Defines the contract for JSON serialization and deserialization.
*   **PrimitiveCodec:** A marker interface that allows the parent serialization engine to identify it as a high-performance codec for a fundamental language type, potentially enabling optimizations.

## Lifecycle & Ownership
-   **Creation:** An instance of BooleanCodec is created by the central CodecRegistry during the application's bootstrap sequence. It is not intended for manual instantiation by client code.
-   **Scope:** The instance is registered and managed by the CodecRegistry, effectively making it a singleton for the entire application lifecycle. It persists for the duration of the game session.
-   **Destruction:** The object is eligible for garbage collection only when the application is shutting down and the primary CodecRegistry is dismantled.

## Internal State & Concurrency
-   **State:** The BooleanCodec is **completely stateless**. It contains no member variables and its methods operate exclusively on the arguments provided. Its behavior is deterministic and consistent across all calls.
-   **Thread Safety:** Due to its stateless nature, this class is inherently **thread-safe**. A single, shared instance can be safely and concurrently invoked by multiple threads performing serialization or deserialization tasks without the need for external locking or synchronization.

## API Surface
The public API provides the core translation and schema generation functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | Boolean | O(1) | Deserializes a BsonValue into a Java Boolean. Throws ClassCastException if the input is not a BsonBoolean. |
| encode(Boolean, ExtraInfo) | BsonValue | O(1) | Serializes a Java Boolean into a BsonBoolean. |
| decodeJson(RawJsonReader, ExtraInfo) | Boolean | O(1) | Deserializes a boolean token from a raw JSON stream. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a corresponding BooleanSchema for data definition and validation purposes. |
| toSchema(SchemaContext, Boolean) | Schema | O(1) | Generates a BooleanSchema and applies a default value if one is provided. |

## Integration Patterns

### Standard Usage
Developers should never invoke BooleanCodec directly. It is used internally by the higher-level serialization engine. The framework is responsible for looking up the correct codec from the registry based on the type of a field and dispatching the serialization call.

```java
// Hypothetical usage by the serialization engine
// A developer would NOT write this code.

// 1. Engine retrieves the registered codec for the Boolean class
Codec<Boolean> booleanCodec = codecRegistry.getCodec(Boolean.class);

// 2. Engine uses the codec to encode a value
BsonValue bsonRepresentation = booleanCodec.encode(true, extraInfo);

// 3. Engine uses the codec to decode a value
Boolean javaRepresentation = booleanCodec.decode(new BsonBoolean(false), extraInfo);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new BooleanCodec()`. The serialization framework relies on a single, managed instance from the CodecRegistry to ensure consistency and performance. Direct instantiation bypasses this system.
-   **Incorrect Type Dispatch:** Passing a non-BsonBoolean value (e.g., BsonString) to the `decode` method will result in a runtime ClassCastException. The calling framework is responsible for ensuring type correctness before dispatch.

## Data Pipeline
The BooleanCodec is a single, critical step in the larger data serialization and deserialization pipelines.

> **Encoding Flow:**
> Java Object Field (boolean) -> Serialization Engine -> **BooleanCodec**.encode -> BsonBoolean -> BSON Document Assembly

> **Decoding Flow:**
> BSON Document -> BsonBoolean Field -> Deserialization Engine -> **BooleanCodec**.decode -> Java Object Field (boolean)

