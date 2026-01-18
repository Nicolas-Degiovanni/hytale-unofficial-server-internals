---
description: Architectural reference for the Codec<T> interface
---

# Codec<T>

**Package:** com.hypixel.hytale.codec
**Type:** Utility Contract / Singleton Provider

## Definition
```java
// Signature
public interface Codec<T> extends RawJsonCodec<T>, SchemaConvertable<T> {
```

## Architecture & Concepts

The Codec interface is the cornerstone of Hytale's data serialization and deserialization framework. It establishes a universal contract for converting between in-memory Java objects and their BSON representations. This system is fundamental for network communication, file storage, and configuration management.

Architecturally, Codec serves two primary purposes:
1.  **Data Transformation:** It defines the bidirectional logic for encoding a Java object of type T into a BsonValue and decoding a BsonValue back into a Java object.
2.  **Schema Definition:** Through its extension of SchemaConvertable, every Codec is capable of generating a formal schema that describes the structure, constraints, and type of data it handles. This enables powerful features like automatic data validation, UI generation for editors, and API documentation.

The interface itself is a contract, but it also acts as a provider for a suite of standard, pre-built codec implementations for primitive types and common Java classes (e.g., STRING, INTEGER, UUID_STRING, PATH). These static instances are designed to be globally accessible and reusable, forming the building blocks for more complex, composite codecs.

The system is designed with BSON as its primary format, but includes a default compatibility layer for handling JSON via the RawJsonCodec interface. This ensures flexibility when interacting with external systems or human-readable configuration files.

## Lifecycle & Ownership

-   **Creation:** Implementations of the Codec interface are designed as stateless singletons. They are either provided as public static final fields on the Codec interface itself (e.g., Codec.STRING) or instantiated once at application startup and injected where needed. They are not intended to be created on a per-operation basis.
-   **Scope:** A Codec's lifecycle is tied to the application. These objects are instantiated once and persist for the entire duration of the client or server process.
-   **Destruction:** There is no manual destruction or cleanup mechanism. Codecs are reclaimed by the Java garbage collector during application shutdown.

## Internal State & Concurrency

-   **State:** A correctly implemented Codec must be **completely stateless and immutable**. It is a pure function that transforms input to output. All contextual information required for a specific operation (such as the path to a field for error reporting) is passed explicitly via the ExtraInfo parameter, not stored as an instance field.
-   **Thread Safety:** Due to their stateless nature, all standard Codec implementations are inherently **thread-safe**. They can be safely shared and used by multiple threads simultaneously without locks or synchronization. This is a critical design feature for high-performance, parallel processing of game data on both the client and server.

**WARNING:** Custom implementations of Codec that introduce mutable state will break this contract and are a source of severe concurrency bugs.

## API Surface

The public contract focuses on the core transformation methods. Deprecated methods are omitted for clarity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(BsonValue, ExtraInfo) | T | O(N) | Deserializes a BsonValue into a Java object of type T. N is the number of nodes in the BSON tree. |
| encode(T, ExtraInfo) | BsonValue | O(N) | Serializes a Java object of type T into a BsonValue. N is the number of fields or elements in the object. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a Schema definition for the data structure handled by this Codec. |
| decodeJson(RawJsonReader, ExtraInfo) | T | O(N) | Deserializes a raw JSON stream. The default implementation buffers this into a BsonValue before decoding. |

## Integration Patterns

### Standard Usage

Developers should almost always use the pre-defined static Codec instances for standard types. For composite objects, these codecs are combined using higher-order codecs (e.g., MapCodec, RecordCodec).

```java
// Standard serialization of a primitive type
BsonValue nameBson = Codec.STRING.encode("PlayerOne", EmptyExtraInfo.EMPTY);

// Standard deserialization
String name = Codec.STRING.decode(nameBson, EmptyExtraInfo.EMPTY);

// Using a functional codec to handle a derived type
Path configPath = Paths.get("./config.json");
BsonValue pathBson = Codec.PATH.encode(configPath, EmptyExtraInfo.EMPTY);
```

### Anti-Patterns (Do NOT do this)

-   **Implementing Stateful Codecs:** A Codec must never store operation-specific data in its fields. This violates thread safety and will lead to data corruption in a multithreaded environment.

    ```java
    // DO NOT DO THIS
    class BadStatefulCodec implements Codec<String> {
        private String lastValue; // BAD: Mutable state
        
        public String decode(BsonValue val, ExtraInfo info) {
            this.lastValue = val.asString().getValue(); // Race condition
            return this.lastValue;
        }
        // ...
    }
    ```

-   **Ignoring The ExtraInfo Parameter:** While older, deprecated methods exist, modern code must pass the ExtraInfo object through all encode and decode calls. Failing to do so will result in useless error messages that lack the context of *where* in a complex data structure the failure occurred.

-   **Re-implementing Standard Codecs:** Do not create a new `new StringCodec()` or `new IntegerCodec()`. Always use the shared static instances like `Codec.STRING` to reduce object allocation and ensure consistent behavior.

## Data Pipeline

The Codec interface is the central processing unit in the engine's data serialization and deserialization pipelines.

> **Serialization Flow:**
> Java Object -> **Codec.encode(object)** -> BsonValue -> BSON Writer -> Network Packet / File on Disk

> **Deserialization Flow:**
> Network Packet / File on Disk -> BSON Reader -> BsonValue -> **Codec.decode(bsonValue)** -> Java Object

