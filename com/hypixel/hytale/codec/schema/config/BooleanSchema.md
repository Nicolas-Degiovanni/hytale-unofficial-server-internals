---
description: Architectural reference for BooleanSchema
---

# BooleanSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class BooleanSchema extends Schema {
```

## Architecture & Concepts
The BooleanSchema class is a data contract used within Hytale's configuration system. It represents a schema definition for a configuration key that must resolve to a boolean value. It is not a service or a manager, but rather a passive data structure that describes validation rules and default values for a specific configuration property.

This class operates as a concrete implementation within a larger, declarative serialization framework. The core of its architecture is the static **CODEC** field, an instance of BuilderCodec. This codec instructs the engine's serialization system on how to read data from a source (like a JSON file) and map it into an instance of BooleanSchema. Specifically, it defines that a schema entry of this type can have an optional field named *default*, which must be a boolean.

BooleanSchema extends the base Schema class, inheriting foundational properties that might be common to all schema types, such as descriptions or validation flags. Its primary role is to provide type-specific constraints for boolean configuration values.

## Lifecycle & Ownership
- **Creation:** Instances of BooleanSchema are not created manually by developers. They are instantiated exclusively by the Hytale codec engine during the deserialization of a configuration schema. The static **CODEC** field is invoked by the engine when it encounters a corresponding data structure in the source file.
- **Scope:** The lifetime of a BooleanSchema object is ephemeral. It exists only as long as the parent configuration schema object that contains it is held in memory, typically during the application's startup or a configuration reload phase.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection as soon as all references to the parsed configuration schema are released. There are no native resources or explicit cleanup procedures.

## Internal State & Concurrency
- **State:** The class holds mutable state in its private *default_* field. This field is populated once during deserialization and is intended to be treated as read-only thereafter.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access during the configuration loading process. Concurrent modification of its state via the setDefault method without external synchronization will lead to undefined behavior.

## API Surface
The primary interaction with this class is through its data, not its methods. The methods exist to support the codec framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getDefault() | Boolean | O(1) | Retrieves the default value specified in the schema definition. Returns null if no default was provided. |

## Integration Patterns

### Standard Usage
A developer does not interact with this Java class directly. Instead, they define a schema in a configuration file which the engine then parses into a BooleanSchema object.

**Example Schema Definition (e.g., in a JSON file):**
```json
{
  "enableCoolFeature": {
    "type": "boolean",
    "description": "Toggles the cool feature on or off.",
    "default": true
  }
}
```
The engine's configuration loader uses BooleanSchema.CODEC to parse the object associated with the *enableCoolFeature* key into a fully hydrated BooleanSchema instance.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new BooleanSchema()`. The object's state is meaningless unless it is populated by the codec system from a valid data source. Manually constructed instances will not be integrated into the engine's configuration validation pipeline.
- **State Mutation After Load:** Do not call `setDefault` on an instance that has been provided by the engine. This modifies the parsed schema in-memory, creating a dangerous divergence from the source configuration file and leading to unpredictable behavior.

## Data Pipeline
The class is a destination point in the configuration loading data pipeline. It transforms raw configuration data into a strongly-typed schema object.

> Flow:
> Raw Config File (e.g., JSON) -> Hytale Codec Deserializer -> **BooleanSchema.CODEC** -> In-Memory BooleanSchema Object -> Configuration System Validation

