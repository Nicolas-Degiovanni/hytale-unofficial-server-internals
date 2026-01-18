---
description: Architectural reference for IntegerSchema
---

# IntegerSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class IntegerSchema extends Schema {
```

## Architecture & Concepts
The IntegerSchema class is a data model that defines a set of validation constraints for an integer value. It is a core component of the Hytale configuration schema system, which is responsible for parsing, validating, and applying configurations defined in external data sources, likely BSON or a similar format.

This class does not contain any validation logic itself. Instead, it serves as a declarative blueprint. A separate validation engine consumes an IntegerSchema instance to check if a given integer conforms to the rules it specifies, such as minimum/maximum bounds, inclusion in an enumeration, or matching a constant value.

Its primary mechanism for instantiation is a static `BuilderCodec`, which deserializes a data structure into a fully-formed IntegerSchema object. This design decouples the schema definition (the data) from the schema implementation (the Java class), allowing for flexible and data-driven configuration.

A notable architectural feature is the use of an `Object` type for boundary fields like *minimum* and *maximum*. This is facilitated by the internal `IntegerOrSchema` codec, which allows these fields to be defined as either a static integer or another Schema. This enables complex, dynamic validation where a boundary can be resolved from another part of the configuration at runtime.

**WARNING:** The internal `IntegerOrSchema` codec is marked as deprecated. This indicates that this mechanism for dynamic schema references may be superseded by a more robust system. Relying on this specific implementation detail is discouraged.

## Lifecycle & Ownership
- **Creation:** IntegerSchema instances are almost exclusively created by the Hytale codec system during the deserialization of configuration files. The static `CODEC` field defines the mapping from a serialized format (e.g., BSON) to the object's fields. Programmatic creation should be limited to the `constant` static factory method for simple, constant-value schemas.
- **Scope:** The lifetime of an IntegerSchema object is tied to the component that loaded it. It typically exists as part of a larger, in-memory tree of Schema objects representing a complete configuration. It is not a global or session-scoped entity.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no native resources or explicit cleanup steps required. It is reclaimed once the root configuration object that references it is no longer in scope.

## Internal State & Concurrency
- **State:** The IntegerSchema is a mutable Plain Old Java Object (POJO). Its state consists of the validation parameters like `minimum`, `maximum`, `enum_`, and `const_`. This state is populated once during deserialization and is not expected to change thereafter.

- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms.

**WARNING:** It is critical to treat IntegerSchema instances as immutable after they are initialized by the codec system. Modifying a schema object from any thread while it is being used by a validator will result in race conditions and undefined behavior. Schemas should be loaded once and then used for reading only.

## API Surface
The primary public contract is the data structure itself, populated by the codec. Direct method calls are infrequent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| constant(int c) | static Schema | O(1) | Factory method to create a simple schema that requires an integer to be a specific constant value. |
| getMinimum() | Object | O(1) | Returns the minimum boundary, which may be an Integer or another Schema. |
| getEnum() | int[] | O(1) | Returns the array of allowed integer values. Returns null if not defined. |
| getConst() | Integer | O(1) | Returns the required constant value. Returns null if not defined. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly. It is used implicitly by the schema and configuration loading systems. A higher-level service will provide access to validated configuration values.

```java
// Hypothetical example of a configuration service using schemas internally
// The developer does not see or create the IntegerSchema instance.

ConfigManager configManager = context.getService(ConfigManager.class);

// The manager loads a file, which the codec system parses into a tree of
// Schema objects, including IntegerSchema for integer fields.
ValidatedConfig playerConfig = configManager.load("player_stats.bson");

// The getInt method internally uses the corresponding IntegerSchema
// to validate and provide a default value if necessary.
int maxHealth = playerConfig.getInt("stats.maxHealth");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IntegerSchema()`. The object will be in an incomplete state and will not function correctly within the validation system. The codec framework is responsible for its construction.

- **Post-Load Modification:** Modifying a schema object after it has been loaded is a severe anti-pattern. This can corrupt the validation logic for all systems sharing that configuration.

```java
// DO NOT DO THIS
// This bypasses the codec and creates an unmanaged object.
IntegerSchema schema = new IntegerSchema();
schema.setMinimum(100);

// DO NOT DO THIS
// This modifies a potentially shared schema, causing side effects.
Schema sharedSchema = config.getSchemaFor("level");
if (sharedSchema instanceof IntegerSchema) {
    ((IntegerSchema) sharedSchema).setMaximum(99); // DANGEROUS
}
```

## Data Pipeline
The IntegerSchema acts as a structured container for rules within the broader data validation pipeline. It holds deserialized data but does not act on it directly.

> Flow:
> BSON Configuration File -> Hytale Codec Engine -> **IntegerSchema Instance** -> Schema Validation Service -> Validation Result / Configured Value

