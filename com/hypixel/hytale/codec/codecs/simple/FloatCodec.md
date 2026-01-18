---
description: Architectural reference for FloatCodec
---

# FloatCodec

**Package:** com.hypixel.hytale.codec.codecs.simple
**Type:** Utility

## Definition
```java
// Signature
public class FloatCodec implements Codec<Float>, RawJsonCodec<Float>, PrimitiveCodec {
```

## Architecture & Concepts
The **FloatCodec** is a foundational component of the Hytale serialization framework. Its sole responsibility is to provide a bidirectional, high-fidelity translation between the Java **Float** type and its serialized representations in both BSON and JSON formats.

This codec is more than a simple type converter. Its key architectural function is to solve the impedance mismatch between Java's floating-point specification and standard data interchange formats. Specifically, it handles the special values **NaN**, **Infinity**, and **-Infinity**, which are valid in Java but have no native representation in JSON.

To preserve data integrity, the **FloatCodec** serializes these special values as strings ("NaN", "Infinity", "-Infinity") while serializing standard numeric values as native numbers. This dual-representation strategy is reflected in the schema it generates, which uses an **anyOf** construct to declare that a value can be either a number or one of the special-case strings. This ensures that data can be round-tripped through the serialization pipeline without loss of precision or meaning.

As a **PrimitiveCodec**, it is automatically registered and utilized by higher-level composite codecs (e.g., for objects or lists) when they encounter a **Float** field.

## Lifecycle & Ownership
- **Creation:** Instances of **FloatCodec** are not intended for manual creation. They are instantiated by a central **CodecRegistry** or a similar service provider during the engine's bootstrap sequence.
- **Scope:** This class is stateless, making its instances reusable and shareable. A single instance typically persists for the entire application session, shared across all threads that require float serialization.
- **Destruction:** The object has no explicit destruction logic. It is eligible for garbage collection when the application shuts down and its parent **CodecRegistry** is dereferenced.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. The **FloatCodec** contains no instance fields and its behavior depends solely on its method inputs. Each call is an independent, atomic operation.
- **Thread Safety:** **Fully thread-safe**. Due to its stateless nature, a single instance can be safely invoked by multiple threads concurrently without any requirement for external locking or synchronization.

## API Surface
The primary API is defined by the **Codec** and **RawJsonCodec** interfaces. The static helper methods are public but are considered implementation details.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Float | O(1) | Deserializes a BsonValue into a Float. Handles both numeric and special string representations. |
| encode(value, extraInfo) | BsonValue | O(1) | Serializes a Float into a BsonDouble. Special values are not supported in BSON and will be encoded as standard doubles. |
| decodeJson(reader, extraInfo) | Float | O(1) | Deserializes a JSON token stream into a Float. Handles both numeric and special string representations. |
| toSchema(context, def) | Schema | O(1) | Generates a data schema describing the dual number/string representation, optionally with a default value. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. It is used implicitly by the overarching serialization service. The framework identifies a **Float** field and delegates its processing to the registered **FloatCodec**.

```java
// Hypothetical high-level serialization service
// The developer only interacts with the top-level CodecEngine.

// Serialization
// The engine internally looks up FloatCodec to handle the 'health' field.
PlayerState state = new PlayerState(100.0f);
BsonValue bson = CodecEngine.getInstance().toBson(state);

// Deserialization
// The engine again uses FloatCodec to deserialize the 'health' field.
PlayerState deserializedState = CodecEngine.getInstance().fromBson(bson, PlayerState.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new FloatCodec()**. The codec framework is responsible for the lifecycle of all codec instances. Circumventing the registry leads to a decoupled and unmanageable system.
- **Using Static Helpers:** Avoid calling the static methods **decodeFloat** or **readFloat** directly. This bypasses the codec framework, ignoring contextual information passed via **ExtraInfo** and breaking potential future enhancements like versioning or logging.

## Data Pipeline
The **FloatCodec** acts as a translation node in the data serialization and deserialization pipelines.

> **Serialization Flow (JSON):**
> Java **Float** -> **FloatCodec** -> JSON Number or "NaN" / "Infinity" String

> **Deserialization Flow (JSON):**
> JSON Number or "NaN" / "Infinity" String -> **RawJsonReader** -> **FloatCodec** -> Java **Float**

