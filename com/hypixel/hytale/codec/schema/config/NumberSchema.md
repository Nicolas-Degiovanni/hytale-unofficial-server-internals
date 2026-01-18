---
description: Architectural reference for NumberSchema
---

# NumberSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class NumberSchema extends Schema {
```

## Architecture & Concepts
The NumberSchema class is a concrete implementation of the Schema contract, designed to define and enforce validation rules for numeric data types. It serves as a core component within the engine's declarative configuration and data validation framework, analogous to the *number* and *integer* types in the JSON Schema specification.

Its primary role is to represent a set of constraints that a numeric value must satisfy. These constraints include range checks (minimum, maximum), value sets (enum), and fixed values (const).

A key architectural feature is its support for dynamic constraints. Fields such as *minimum* and *maximum* are not limited to static double values; they can also be references to other Schema objects. This allows for complex validation scenarios where, for example, a value's minimum bound is determined by another configuration value resolved at runtime. This is facilitated internally by the deprecated DoubleOrSchema codec, which can decode a BSON field as either a raw number or a nested Schema object.

The static CODEC field, an instance of BuilderCodec, is central to its function. It declaratively maps serialized BSON fields (e.g., "minimum", "const") to the corresponding Java fields of a NumberSchema instance, forming the bridge between on-disk schema definitions and their in-memory object representation.

## Lifecycle & Ownership
- **Creation:** NumberSchema instances are primarily created by the framework's `BuilderCodec` during the deserialization of a larger schema definition from a BSON source. They can also be instantiated programmatically via the public constructor or the `constant` static factory method for testing or dynamic schema construction.
- **Scope:** The lifetime of a NumberSchema object is transient and is strictly bound to the lifecycle of the root schema configuration that contains it. It is not a globally managed or session-scoped service.
- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup as soon as the parent schema is no longer referenced. No manual destruction or resource release is necessary.

## Internal State & Concurrency
- **State:** Mutable. NumberSchema is a Plain Old Java Object (POJO) whose state is composed of its validation constraint fields (e.g., minimum, maximum, enum_). The state is intended to be fully configured upon creation and then treated as immutable.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It is designed to be constructed and configured on a single thread (typically during application bootstrap or asset loading). Once a NumberSchema is part of a shared configuration, it **must** be treated as immutable. Any concurrent modification will result in race conditions and unpredictable validation behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setMinimum(double) | void | O(1) | Sets a static, numeric minimum bound. |
| setMinimum(Schema) | void | O(1) | Sets a dynamic minimum bound resolved by another schema. |
| setMaximum(double) | void | O(1) | Sets a static, numeric maximum bound. |
| setMaximum(Schema) | void | O(1) | Sets a dynamic maximum bound resolved by another schema. |
| setEnum(double[]) | void | O(1) | Restricts the valid value to one of the numbers in the array. |
| setConst(Double) | void | O(1) | Restricts the valid value to a single, constant number. |
| constant(double) | static Schema | O(1) | Factory method to create a simple schema that only allows one specific value. |

## Integration Patterns

### Standard Usage
A developer typically defines a schema declaratively in a configuration file. Programmatic construction is used for dynamic generation or testing.

```java
// Programmatically create a schema for a percentage value (0.0 to 1.0)
NumberSchema percentageSchema = new NumberSchema();
percentageSchema.setMinimum(0.0);
percentageSchema.setMaximum(1.0);

// This schema object would then be used by a validator service.
// validator.validate(percentageSchema, 0.75); // would pass
// validator.validate(percentageSchema, -1.0); // would fail
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Modifying a NumberSchema object after it has been registered with a validator or shared across systems is extremely dangerous. This breaks the assumption of immutability and can lead to inconsistent validation results across the application.

- **Concurrent Access:** Do not share a NumberSchema instance between threads if any thread might modify it. All mutations must be completed before the object is published for use by other threads.

## Data Pipeline
The NumberSchema participates in two primary data flows: schema deserialization and data validation.

**1. Schema Deserialization Flow**
> Flow:
> BSON Document -> `BuilderCodec` -> **NumberSchema Instance** -> Schema Registry

**2. Data Validation Flow**
> Flow:
> Input BsonValue -> Validator Service -> **NumberSchema Instance** -> Validation Result (Pass/Fail)

