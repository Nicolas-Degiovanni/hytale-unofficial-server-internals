---
description: Architectural reference for DoubleCodec
---

# DoubleCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class DoubleCodec implements Codec<Double>, RawJsonCodec<Double>, PrimitiveCodec {
```

## Architecture & Concepts
The DoubleCodec is a fundamental, low-level component within the Hytale serialization framework. Its sole responsibility is the bidirectional translation between the in-memory Java Double type and its serialized representations in BSON and JSON formats.

As a primitive codec, it serves as a foundational building block for more complex object serializers. The central CodecEngine automatically discovers and delegates to this implementation whenever it encounters a Double field during a serialization or deserialization operation.

A key architectural feature is its explicit handling of non-standard floating-point values. It correctly serializes and deserializes special string representations for **NaN**, **Infinity**, and **-Infinity**. This design choice ensures data fidelity for scientific or mathematical computations where such values are common, a capability often lost in standard, less-permissive JSON parsers.

Furthermore, its implementation of the Codec interface includes the ability to generate a corresponding data schema, which is critical for asset validation, network protocol definition, and developer tooling.

## Lifecycle & Ownership
- **Creation:** A single instance of DoubleCodec is instantiated by the central CodecRegistry during the engine's bootstrap sequence. It is not intended for manual creation by developers.
- **Scope:** The registered instance is effectively a singleton that persists for the entire application session. Its stateless nature allows it to be shared globally without conflict.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the CodecRegistry is torn down, typically during a controlled application shutdown.

## Internal State & Concurrency
- **State:** The DoubleCodec is **immutable and stateless**. It contains no instance fields, and its transformation logic is purely functional, depending only on the input arguments.
- **Thread Safety:** This class is inherently thread-safe. Due to its stateless design, a single instance can be safely and concurrently invoked by multiple threads performing serialization or deserialization tasks without requiring any external locks or synchronization.

## API Surface
The public API provides the core serialization, deserialization, and schema generation contracts required by the parent CodecEngine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Double | O(1) | Deserializes a BsonValue into a Java Double. Correctly interprets numeric types and special string constants like "NaN". |
| encode(t, extraInfo) | BsonValue | O(1) | Serializes a Java Double into a BsonDouble. |
| decodeJson(reader, extraInfo) | Double | O(1) | Deserializes a double value from a raw JSON stream. |
| toSchema(context) | Schema | O(1) | Generates a standard NumberSchema definition for a double. |
| toSchema(context, def) | Schema | O(1) | Generates a NumberSchema and attaches a default value, unless the default is NaN or infinite. |

## Integration Patterns

### Standard Usage
Developers do not interact with DoubleCodec directly. It is used transparently by higher-level serialization services like a CodecEngine or a data binder. The engine identifies a Double field and automatically dispatches the operation to the registered DoubleCodec instance.

```java
// The CodecEngine internally selects and delegates to DoubleCodec
// for fields of type Double. A developer would not invoke it directly.

// Hypothetical high-level usage:
CodecEngine engine = context.getService(CodecEngine.class);

// ENCODING: The engine finds the registered DoubleCodec for 3.14159
BsonValue bson = engine.encode(3.14159);

// DECODING: The engine is instructed to produce a Double and uses
// the DoubleCodec to perform the conversion.
Double value = engine.decode(bson, Double.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance via `new DoubleCodec()`. The serialization system relies on a single, globally registered instance to function correctly. Manually creating instances bypasses the registry and can lead to unpredictable behavior.
- **Subclassing:** This class is a final implementation and is not designed for extension. Its logic is atomic and complete for the Java Double type.

## Data Pipeline
The DoubleCodec acts as a specific transformation step within a larger data serialization or deserialization pipeline.

> **BSON Deserialization Flow:**
> BSON Byte Stream -> BsonValue Parser -> **DoubleCodec.decode** -> Java Double

> **BSON Serialization Flow:**
> Java Double -> **DoubleCodec.encode** -> BsonDouble -> BSON Byte Stream

> **JSON Deserialization Flow:**
> JSON Text Stream -> RawJsonReader -> **DoubleCodec.decodeJson** -> Java Double

