---
description: Architectural reference for Schema
---

# Schema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Data Model (Transient)

## Definition
```java
// Signature
public class Schema {
```

## Architecture & Concepts
The Schema class is the core Java representation of a Hytale configuration file's structure, conforming to a superset of the JSON Schema specification. It is not a service or manager, but rather a data model that defines the validation rules, data types, constraints, and engine-specific metadata for assets and configuration objects.

Architecturally, Schema and its subclasses form the foundation of the engine's data-driven configuration system. Its primary role is to be the target for deserialization from BSON (Binary JSON) documents. This process is managed by a set of powerful, statically defined Codec objects.

- **Polymorphic Deserialization:** The static field **CODEC** is an ObjectCodecMapCodec. This is a specialized codec that enables polymorphism. When deserializing a BSON document, it inspects the **type** field (e.g., "object", "string", "integer") to determine which concrete subclass of Schema (e.g., ObjectSchema, StringSchema) to instantiate. This allows the system to parse a complex, nested schema document into a corresponding tree of typed Java objects.

- **Declarative Mapping:** The static **BASE_CODEC** field is a BuilderCodec. This object declaratively maps BSON field names (e.g., "$id", "title", "anyOf") to the fields of the Schema Java class. This pattern avoids boilerplate manual parsing logic and centralizes the serialization contract in one place.

- **Engine-Specific Metadata:** The class includes numerous fields prefixed with **hytale**, such as hytaleAssetRef and the nested HytaleMetadata class. These fields are Hytale-specific extensions to the JSON Schema standard, used to provide hints to the Hytale editor for UI rendering, asset linking, property inheritance, and other editor-time features.

The static **init** method acts as a registry bootstrap. It must be called during application startup to register all concrete Schema types with the master **CODEC**, making them available for polymorphic deserialization.

### Lifecycle & Ownership
- **Creation:** Schema objects are almost exclusively instantiated by the codec system during the deserialization of a BSON or JSON file. Programmatic creation is possible via the public constructor or static factory methods (e.g., Schema.ref, Schema.anyOf) for in-memory schema generation or testing.
- **Scope:** The lifetime of a Schema object is transient. It is typically loaded to validate a configuration file or to provide metadata to an editor tool, and is then eligible for garbage collection once the operation is complete. It does not persist across sessions.
- **Destruction:** Managed entirely by the Java garbage collector. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** The Schema class is a mutable data container. Its fields hold the deserialized schema definition and can be modified at runtime via public setters. It does not perform any significant caching, with the minor exception of the getEnumDescriptions method which can derive its value from markdownEnumDescriptions if necessary.

- **Thread Safety:** **This class is not thread-safe.** It is a plain data object with no internal synchronization. Concurrent modification of a Schema instance from multiple threads will lead to race conditions and undefined behavior. If a Schema object must be shared between threads, all access must be synchronized externally.

## API Surface
The public API consists primarily of standard getters and setters for its properties. The most architecturally significant symbols are static.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | ObjectCodecMapCodec | O(1) | The master polymorphic codec for deserializing any Schema object. |
| BASE_CODEC | BuilderCodec | O(1) | The codec defining field mappings for the base Schema class. |
| init() | static void | O(N) | Registers all known Schema subclasses into the master CODEC. **CRITICAL:** Must be called at startup. |
| ref(String) | static Schema | O(1) | Factory method to create a schema that references another schema definition. |
| anyOf(Schema...) | static Schema | O(1) | Factory method to create a schema that validates against any of the provided sub-schemas. |
| getHytale(...) | HytaleMetadata | O(1) | Retrieves Hytale-specific metadata, lazy-instantiating the container if it does not exist. |

## Integration Patterns

### Standard Usage
Developers should never need to interact with the Schema class directly for standard configuration loading. Interaction is mediated through a higher-level asset or configuration manager that uses the codec system internally. Programmatic creation is for advanced use cases like dynamic schema generation.

```java
// Hypothetical example of how the system uses the Schema.CODEC
// A developer would typically call a higher-level function like `assetManager.loadConfig`.

// 1. A BSON document is read from a file.
BsonDocument bsonData = readBsonFromFile("my_asset.json");

// 2. The codec system is invoked to decode the BSON into a Schema object.
// The CODEC automatically selects the correct subclass (e.g., ObjectSchema).
ObjectSchema schema = (ObjectSchema) Schema.CODEC.decode(bsonData);

// 3. The engine can now use the schema object to validate other assets
// or configure editor UI.
if (schema.getProperties().containsKey("position")) {
    // Logic based on schema definition
}
```

### Anti-Patterns (Do NOT do this)
- **Forgetting Initialization:** Attempting to use Schema.CODEC to deserialize data before `Schema.init()` has been called will result in runtime exceptions, as the codec will not know how to map type strings like "object" or "string" to their corresponding Java classes.
- **Concurrent Modification:** Modifying a Schema object from one thread while it is being read by another. This will corrupt state and is a severe bug. Always ensure thread-safe access patterns if sharing instances.
- **Misusing References:** Using `setRef` to create circular dependencies without a proper resolution strategy can lead to stack overflow errors during validation or processing.

## Data Pipeline
The Schema class is a key component in the data serialization and deserialization pipeline for Hytale's configuration files.

> **Deserialization Flow:**
> BSON Document (from .json file) -> **Schema.CODEC.decode()** -> Polymorphic Instantiation (e.g., ObjectSchema) -> **Schema Object Tree**

> **Serialization Flow:**
> **Schema Object Tree** -> **Schema.CODEC.encode()** -> BSON Document (for writing to file)

