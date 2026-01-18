---
description: Architectural reference for StringIntegerCodec
---

# StringIntegerCodec

**Package:** com.hypixel.hytale.codec.codecs
**Type:** Singleton

## Definition
```java
// Signature
@Deprecated
public class StringIntegerCodec implements Codec<Integer> {
```

## Architecture & Concepts
The StringIntegerCodec is a specialized data format adapter within the codec subsystem. Its primary architectural function is to provide resilience and backward compatibility when deserializing data that may have inconsistent typing. Specifically, it handles cases where a numerical integer value is incorrectly encoded as a string (e.g., "123") instead of a native integer type (e.g., 123) in BSON or JSON payloads.

This component embodies the *Tolerant Reader* pattern. It is designed to gracefully handle and normalize malformed or legacy data from external sources without causing a hard failure in the deserialization pipeline.

**WARNING:** The Deprecated annotation on this class is a critical signal. This codec is considered a legacy component or a temporary workaround. Its existence implies a known data integrity issue in an upstream system. New systems should enforce strict schema validation and avoid relying on this type of lenient decoding.

## Lifecycle & Ownership
- **Creation:** The singleton INSTANCE is instantiated by the JVM during class loading. It is not created by any application-level factory or dependency injection container.
- **Scope:** As a static singleton, its lifecycle is tied to the application's lifecycle. It persists from the moment the class is loaded until the application terminates.
- **Destruction:** The object is eligible for garbage collection only upon application shutdown. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its static fields are final. All operations are pure functions of their inputs.
- **Thread Safety:** The StringIntegerCodec is inherently **thread-safe**. Its stateless nature ensures that it can be safely shared and invoked concurrently by multiple threads in the network or data processing layers without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | Integer | O(N) | Decodes a BsonValue into an Integer. Handles both native number types and string-encoded integers. N is the length of the string. |
| encode(t, extraInfo) | BsonValue | O(1) | Encodes an Integer into a standard BsonInt32. It does not encode to a string, enforcing correct outbound data format. |
| decodeJson(reader, extraInfo) | Integer | O(N) | Decodes a JSON token from a stream into an Integer. Handles both JSON numbers and string-quoted numbers. N is the length of the token. |
| toSchema(context) | StringSchema | O(1) | Generates a schema definition for this type, indicating a string that must match an integer pattern. Used for schema validation systems. |

## Integration Patterns

### Standard Usage
This codec is not intended for direct invocation. It should be registered within a central CodecRegistry or a similar mapping mechanism. The deserialization engine will then automatically select and use this codec when encountering a field that requires an Integer but may receive a string.

```java
// Example of registering the codec (conceptual)
CodecRegistry registry = new CodecRegistry();

// The registry would be configured to use this for specific fields
// or as a fallback for Integer types.
registry.registerFallback(Integer.class, StringIntegerCodec.INSTANCE);

// Subsequent deserialization automatically uses the codec
MyObject obj = deserializer.decode(someBson, MyObject.class);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new StringIntegerCodec()`. The provided static `StringIntegerCodec.INSTANCE` must be used to maintain its singleton contract.
- **Masking Systemic Errors:** Do not use this codec to hide persistent data integrity problems in new services. If a service is consistently writing integers as strings, the producing service must be fixed. This codec is for compatibility, not for enabling poor data hygiene.
- **Encoding Logic:** Do not assume the `encode` method will produce a string. This codec is asymmetric; it reads leniently but writes strictly, always encoding integers to the correct BsonInt32 type to prevent the propagation of bad data.

## Data Pipeline

The primary role of this component is during the decoding (deserialization) phase of a data pipeline.

> **Decoding Flow:**
> Network Packet -> BSON/JSON Parser -> **StringIntegerCodec.decode()** -> In-memory Integer -> Game Logic

> **Encoding Flow:**
> Game Logic -> In-memory Integer -> **StringIntegerCodec.encode()** -> BsonInt32 -> BSON Serializer -> Network Packet

