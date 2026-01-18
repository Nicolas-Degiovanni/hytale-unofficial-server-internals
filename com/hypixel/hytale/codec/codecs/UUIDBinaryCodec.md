---
description: Architectural reference for UUIDBinaryCodec
---

# UUIDBinaryCodec

**Package:** com.hypixel.hytale.codec.codecs
**Type:** Utility

## Definition
```java
// Signature
public class UUIDBinaryCodec implements Codec<UUID> {
```

## Architecture & Concepts

The UUIDBinaryCodec is a specialized component within the Hytale Codec Framework responsible for the bidirectional translation of Java UUID objects. It serves as a concrete implementation of the generic Codec interface, acting as the definitive authority for serializing and deserializing UUIDs across the entire engine.

This class is a critical data-interchange component, bridging the gap between the high-level, in-memory representation of a UUID and its low-level, on-the-wire or on-disk formats. It supports two primary encodings:

1.  **BSON Binary:** A highly efficient, 16-byte binary representation compliant with the BSON specification's standard UUID subtype. This is the preferred format for internal communication and database storage due to its compactness.
2.  **JSON:** A text-based representation for contexts where human readability or web compatibility is required. The codec can parse both a simple Base64-encoded string and the more verbose BSON-extended JSON object format (e.g., `{ "$binary": "...", "$type": "04" }`).

By conforming to the Codec interface, this class allows the broader serialization system to handle UUIDs polymorphically. A higher-level service, such as a CodecRegistry, discovers and delegates to this implementation whenever a UUID needs to be processed, decoupling the core system from the specific details of UUID encoding.

## Lifecycle & Ownership

-   **Creation:** The UUIDBinaryCodec is stateless and designed to be instantiated once by a central service registry or dependency injection container during application bootstrap. It is typically registered within a map of type-to-codec handlers.
-   **Scope:** An instance of this codec is expected to be a long-lived singleton, persisting for the entire application lifecycle. Its stateless nature allows a single instance to be shared safely across the entire system.
-   **Destruction:** The object requires no explicit cleanup and is garbage collected when its parent registry is destroyed, typically during application shutdown.

## Internal State & Concurrency

-   **State:** This class is **stateless**. It contains no instance fields and all of its methods are pure functions whose output depends solely on their input arguments. It does not cache any data.
-   **Thread Safety:** The UUIDBinaryCodec is **unconditionally thread-safe**. Due to its stateless design, a single instance can be invoked concurrently from multiple threads without any risk of race conditions or data corruption. No external locking or synchronization is required when using this class.

## API Surface

The primary API is the contract defined by the Codec interface. The public static helper methods are considered implementation details and should not be called directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | UUID | O(1) | Deserializes a BSON binary value into a UUID. Throws CodecException if the BSON subtype is not UUID_STANDARD. |
| encode(uuid, extraInfo) | BsonValue | O(1) | Serializes a Java UUID into a BSON binary value with the standard subtype. |
| decodeJson(reader, extraInfo) | UUID | O(N) | Deserializes a JSON string or object into a UUID. N is the length of the string. Throws IOException on malformed input. |
| toSchema(context) | Schema | O(1) | Generates a schema definition describing a Base64-encoded UUID for validation or documentation purposes. |

## Integration Patterns

### Standard Usage

This codec is not intended for direct use. It is a framework component that is automatically invoked by a higher-level serialization service. Developers interact with the service, which in turn delegates to this codec.

```java
// Hypothetical usage via a parent serialization service
// The service internally looks up and uses UUIDBinaryCodec.

UUID entityId = UUID.randomUUID();
BsonValue bsonId = codecService.toBson(entityId);

// ... network transmission ...

UUID decodedId = codecService.fromBson(bsonId, UUID.class);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not create instances with `new UUIDBinaryCodec()`. The codec framework manages the lifecycle of this object. Rely on the central registry to provide the instance.
-   **Misusing Static Helpers:** The public static methods like `uuidFromBytes` and `uuidFromHex` exist to support the codec's implementation. Calling them directly bypasses the codec framework.
-   **Incorrect Data Format:** The `decode` method strictly expects a BsonBinary with the `UUID_STANDARD` subtype (0x04). Passing any other binary subtype will result in a runtime CodecException.

**WARNING:** The static method `uuidFromHex` is misnamed. It decodes a **Base64** string, not a hexadecimal string. Relying on this method directly can lead to critical data corruption bugs if the name is taken at face value.

## Data Pipeline

The UUIDBinaryCodec acts as a translation step in a larger data serialization or deserialization pipeline.

> **Encoding Flow:**
> Java `UUID` Object -> **UUIDBinaryCodec.encode()** -> `BsonBinary` -> Network Buffer / Database

> **Decoding Flow:**
> Network Buffer / Database -> `BsonBinary` -> **UUIDBinaryCodec.decode()** -> Java `UUID` Object

