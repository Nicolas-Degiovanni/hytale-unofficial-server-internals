---
description: Architectural reference for EqualValidator
---

# EqualValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient Utility

## Definition
```java
// Signature
public class EqualValidator<T extends Comparable<T>> implements Validator<T> {
```

## Architecture & Concepts
The EqualValidator is a concrete implementation of the Validator interface, designed to enforce a strict equality constraint on a configuration value. It is a fundamental component within the Hytale Codec and Schema system, responsible for ensuring that a given piece of data matches a predefined constant value.

Its architectural significance lies in its dual-function design:
1.  **Runtime Validation:** The `accept` method performs the actual validation check during the data deserialization and validation pipeline.
2.  **Schema Declaration:** The `updateSchema` method translates this runtime rule into a declarative constraint within a Schema object, typically by setting the `const` property. This allows the system to generate a complete and accurate JSON Schema, enabling static analysis, client-side validation, and automated documentation.

This class acts as a bridge between imperative validation logic and the declarative schema definition, ensuring they remain synchronized.

## Lifecycle & Ownership
- **Creation:** An EqualValidator is instantiated programmatically whenever a specific equality rule is needed. It is not a managed service or singleton. It is typically created and passed to a schema builder or a validation rule set.
- **Scope:** The lifetime of an EqualValidator instance is tied directly to the Schema or validation chain that contains it. It is a short-lived object.
- **Destruction:** The object is eligible for garbage collection as soon as its owning Schema or validation configuration is no longer referenced. No explicit cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single `final` field, `value`, which holds the constant to compare against. This value is set once at construction and cannot be modified thereafter.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable design, a single EqualValidator instance can be safely shared and executed by multiple threads concurrently without requiring any external locking or synchronization.

## API Surface
The public contract is defined by the Validator interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T o, ValidationResults results) | void | O(1) | Executes the validation logic. If the input object `o` is not equal to the configured value, a failure is registered in the `results` object. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Modifies the target Schema to reflect the equality constraint. Throws IllegalArgumentException if the target schema already contains an `allOf` definition. |

## Integration Patterns

### Standard Usage
The EqualValidator is intended to be used as part of a larger schema definition process, often through a builder or factory pattern.

```java
// A conceptual example of adding an equality validator to a schema builder
StringSchema nameSchema = new StringSchema();

// The validator ensures the string must be exactly "default_user"
Validator<String> nameValidator = new EqualValidator<>("default_user");

// The validator is applied to the schema
nameValidator.updateSchema(schemaContext, nameSchema);

// At runtime, the validator checks the input data
ValidationResults results = new ValidationResults();
nameValidator.accept("some_other_user", results); // This will fail
```

### Anti-Patterns (Do NOT do this)
- **Null Comparison Value:** The constructor is annotated with Nonnull. Passing a null value for the comparison target will result in a runtime failure and is not a supported use case.
- **Incompatible Types:** While the `updateSchema` method contains runtime type checks, developers should not rely on them. Do not attempt to apply an EqualValidator for a String to an IntegerSchema. This indicates a flaw in the schema construction logic.
- **Ignoring Schema Updates:** Failing to call `updateSchema` after creating a validator will lead to a critical system inconsistency: the data will be validated at runtime, but the generated schema will not reflect the constraint, misleading consumers of the schema.

## Data Pipeline
The EqualValidator operates at a specific point within two primary data flows: schema generation and data validation.

**Schema Generation Flow:**
> Schema Builder -> **EqualValidator.updateSchema()** -> Mutated Schema Object -> Final JSON Schema

**Data Validation Flow:**
> Raw Data (JSON, etc.) -> Codec Deserializer -> Validation Chain -> **EqualValidator.accept()** -> ValidationResults -> Final Typed Object or Error

