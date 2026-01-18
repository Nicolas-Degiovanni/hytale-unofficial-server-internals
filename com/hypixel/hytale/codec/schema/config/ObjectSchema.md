---
description: Architectural reference for ObjectSchema
---

# ObjectSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class ObjectSchema extends Schema {
```

## Architecture & Concepts
The ObjectSchema class is a data structure that represents a structural contract for a complex object within the Hytale Codec framework. It is the primary mechanism for defining, validating, and describing nested object configurations, analogous to the *object* type in JSON Schema.

This class does not contain validation or serialization logic itself. Instead, it serves as a declarative model. The actual processing logic is handled by the Hytale Codec engine, which interprets an ObjectSchema instance to perform its operations. The static final field **CODEC** is the most critical component; it is a BuilderCodec that defines how an ObjectSchema is itself deserialized from a configuration source. This decouples the data representation (ObjectSchema) from the serialization mechanics (BuilderCodec).

ObjectSchema is a foundational, recursive component. Its *properties* map contains other Schema instances, allowing for the definition of deeply nested and complex data hierarchies.

## Lifecycle & Ownership
- **Creation:** ObjectSchema instances are not intended to be created directly by developers. They are instantiated by the Hytale Codec engine during the deserialization of a schema definition file. The static **CODEC** field is invoked by the engine, which populates a new ObjectSchema instance field-by-field.
- **Scope:** The lifetime of an ObjectSchema is bound to the root configuration object that contains it. Typically, schemas are loaded once during application or world initialization and persist in memory for the duration of that context.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup once the root schema definition is no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The internal state is **highly mutable**. ObjectSchema is fundamentally a Plain Old Java Object (POJO) with public setters for its definitional fields. This design facilitates construction by the BuilderCodec system.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms.

    **WARNING:** Schema objects must be treated as effectively immutable after initial configuration by the codec system. Modifying a shared ObjectSchema instance from multiple threads at runtime will lead to race conditions and non-deterministic behavior in the validation and serialization systems that rely on it. All schema definitions should be finalized before being used by concurrent processes.

## API Surface
The public API is designed for configuration by the framework. Direct invocation by client code is rare and discouraged.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setProperties(Map) | void | O(1) | Assigns the map of named properties and their corresponding schemas. |
| setAdditionalProperties(boolean) | void | O(1) | Sets a boolean flag to permit or forbid properties not explicitly defined. |
| setAdditionalProperties(Schema) | void | O(1) | Provides a schema to which all additional, non-defined properties must conform. |
| setPropertyNames(StringSchema) | void | O(1) | Assigns a schema that all property keys themselves must validate against. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with ObjectSchema instances directly. Instead, they define the schema declaratively in a configuration file (e.g., JSON) which is then loaded by the engine. The most common interaction is with the *codec* that *uses* the schema, not the schema object itself.

A valid, but rare, use case might involve programmatic inspection of a loaded schema.

```java
// Assume 'schemaRegistry' is a service that holds pre-loaded schemas
Schema entitySchema = schemaRegistry.getSchema("hytale:mob_definition");

if (entitySchema instanceof ObjectSchema) {
    ObjectSchema objectSchema = (ObjectSchema) entitySchema;
    // Inspect the schema, for example to build a dynamic UI editor
    Map<String, Schema> properties = objectSchema.getProperties();
    System.out.println("Entity has properties: " + properties.keySet());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ObjectSchema()`. The object is not valid until its fields are populated in a specific order by the framework's codec. Bypassing the codec can result in a partially initialized object that will cause runtime exceptions.
- **Runtime Modification:** Do not get a schema from a registry and call its setters. Schemas are often cached and shared across the application. Modifying a schema after it has been loaded will have unpredictable side effects on all systems using that schema.

## Data Pipeline
ObjectSchema is a blueprint, not an active participant in a data stream. Its primary role is during the initialization of the codec system.

> Flow (Schema Deserialization):
> Configuration File (e.g., entity_definition.json) -> Raw Data Stream -> Hytale Codec Engine -> **BuilderCodec<ObjectSchema>** -> **ObjectSchema Instance** -> Schema Registry Cache

