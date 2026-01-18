---
description: Architectural reference for RangeRefValidator
---

# RangeRefValidator<T extends Comparable<T>>

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class RangeRefValidator<T extends Comparable<T>> implements Validator<T> {
```

## Architecture & Concepts
The RangeRefValidator is a specialized component within the Hytale Codec and Schema system. Its primary function is not to validate data directly, but to dynamically configure a schema with range constraints by referencing values from elsewhere in a data context.

This class embodies the "configuration as data" principle. Instead of hardcoding minimum and maximum values, it uses string-based pointers (e.g., JSON Pointers) to locate these boundary values at schema generation time. This allows for the creation of highly dynamic and context-aware validation rules where, for example, the valid range for one field can be defined by the value of another field.

Architecturally, it acts as a schema *mutator*. It is invoked during a schema-building phase to inject `minimum`, `maximum`, `exclusiveMinimum`, or `exclusiveMaximum` properties into a target NumberSchema or IntegerSchema.

**WARNING:** A critical design aspect is the empty implementation of the `accept` method. This signals that RangeRefValidator does not participate in the runtime validation phase of the `Validator` interface. Its sole responsibility is fulfilled during the schema setup phase via the `updateSchema` method.

## Lifecycle & Ownership
- **Creation:** Instantiated programmatically by a higher-level schema or validation factory. This factory is responsible for parsing validation rules from a definition file and constructing the appropriate Validator instances. A developer will almost never instantiate this class directly.
- **Scope:** Short-lived. Its existence is scoped to the construction and configuration of a single Schema object. Once its `updateSchema` method has been called, it typically has no further purpose and is eligible for garbage collection.
- **Destruction:** Managed by the Java garbage collector. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state is **immutable**. The `minPointer`, `maxPointer`, and `inclusive` fields are final and set only during construction. The object holds no other mutable state or caches.
- **Thread Safety:** The instance is inherently thread-safe due to its immutable state. It can be safely shared and used across multiple threads to configure different schema objects. However, the thread safety of the overall operation depends on the external `Schema` object being mutated.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RangeRefValidator(min, max, inclusive) | constructor | O(1) | Creates a new validator with pointers to the min/max values. |
| accept(T, ValidationResults) | void | O(1) | No-op. Fulfills the Validator interface but performs no action. |
| updateSchema(SchemaContext, Schema) | void | O(1) | The core method. Modifies the target Schema with range constraints. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use. It is designed to be managed by a validation framework. The conceptual usage below illustrates its role in a schema-building process.

```java
// Conceptual Example: How the framework uses the validator
// 1. A validator is created from a config rule
RangeRefValidator validator = new RangeRefValidator("/config/minHealth", "/config/maxHealth", true);

// 2. A schema for a numeric field is being built
IntegerSchema healthSchema = new IntegerSchema();
SchemaContext context = getSchemaContext(); // Assume context is available

// 3. The validator is applied to the schema
validator.updateSchema(context, healthSchema);

// 4. The healthSchema is now configured with:
// minimum: { "$data": "/config/minHealth" }
// maximum: { "$data": "/config/maxHealth" }
```

### Anti-Patterns (Do NOT do this)
- **Calling `accept`:** Do not invoke the `accept` method. It is intentionally empty and calling it has no effect. This indicates a misunderstanding of the class's purpose, which is schema configuration, not data validation.
- **Expecting Runtime Validation:** Do not rely on this class to perform validation at runtime. It only prepares a schema; a separate, schema-aware validation engine is responsible for the actual data validation.
- **Applying to Non-Numeric Schemas:** While the code logs a warning, applying this validator to a non-numeric schema (e.g., StringSchema) will have no effect. The schema will not be modified, and no error will be thrown.

## Data Pipeline
RangeRefValidator operates during the *design-time* or *configuration-time* phase of the data pipeline. It does not process live data itself.

> Flow:
> Validation Rule Definition (e.g., JSON) -> Schema Factory -> **RangeRefValidator Instantiation** -> `updateSchema` call -> Modified Schema Object -> Schema-based Validator Engine -> Runtime Data Validation

