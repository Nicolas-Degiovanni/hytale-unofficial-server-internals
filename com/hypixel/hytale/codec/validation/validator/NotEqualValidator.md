---
description: Architectural reference for NotEqualValidator
---

# NotEqualValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient Utility

## Definition
```java
// Signature
public class NotEqualValidator<T extends Comparable<T>> implements Validator<T> {
```

## Architecture & Concepts
The NotEqualValidator is a concrete implementation of the Validator interface within the Hytale Codec and Schema system. Its purpose is to enforce a constraint that a given value must not be equal to a predefined constant. This component is a fundamental primitive in the engine's declarative validation framework.

It operates in two distinct, yet related, capacities:

1.  **Runtime Validation:** During data processing, the `accept` method is invoked to perform a direct comparison. It acts as a gate, failing the validation process if the input data matches the forbidden value.
2.  **Schema Generation:** The `updateSchema` method modifies a formal Schema definition to include the "not equal" rule. This is critical for systems that rely on static schema analysis, such as generating client-side validation, exporting to standard formats like JSON Schema, or powering editor tool-tips.

By bridging runtime logic with declarative schema modification, NotEqualValidator ensures that a single validation rule is consistently represented and enforced across the entire Hytale platform.

## Lifecycle & Ownership
-   **Creation:** Instances are created dynamically by higher-level validation builders or schema factories. A developer typically specifies a "not equal" rule through a configuration API, which in turn instantiates this class. Direct construction is uncommon and generally discouraged.
-   **Scope:** Short-lived and task-specific. An instance exists only for the duration of a single validation pass or a single schema-building operation. It is not registered as a service and holds no persistent state.
-   **Destruction:** The object is immediately eligible for garbage collection once the operation that created it completes. It requires no explicit resource management or cleanup.

## Internal State & Concurrency
-   **State:** **Immutable**. The validator's state consists of a single `final` field, `value`, which is set at construction. Once created, its internal state cannot be changed.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable design, a single instance can be safely used across multiple threads without any external locking or synchronization. This allows it to be used in parallel validation scenarios.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, ValidationResults) | void | O(1) | Executes the validation logic. If the input value equals the configured value, a failure is recorded in the provided ValidationResults object. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies the target Schema to declaratively represent the "not equal" constraint. Throws IllegalArgumentException if the target schema type is unsupported or if its `allOf` property is already populated. |

## Integration Patterns

### Standard Usage
This class is an internal component of the validation framework and is not typically used directly. It is invoked by higher-level systems that build validation rulesets.

```java
// The framework constructs the validator based on a declarative rule.
// This example simulates how the framework would use the class.

Validator<String> notAdminValidator = new NotEqualValidator<>("admin");
ValidationResults results = new ValidationResults();

// The framework passes an input value to the validator.
notAdminValidator.accept("guest", results);
// results.hasFailed() is now false.

notAdminValidator.accept("admin", results);
// results.hasFailed() is now true.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new NotEqualValidator()`. Define validation rules through the appropriate schema configuration or builder APIs to ensure they are correctly integrated into the entire system.
-   **Schema Mismatch:** Passing a Schema object to `updateSchema` whose type does not correspond to the validator's generic type T (e.g., passing an IntegerSchema to a `NotEqualValidator<String>`) will lead to runtime exceptions.
-   **Ignoring Schema Updates:** Using this validator for runtime checks via `accept` but failing to integrate it into the schema generation process via `updateSchema` creates a system inconsistency. External tools and clients will be unaware of the validation rule.

## Data Pipeline
The NotEqualValidator participates in two primary data flows within the engine.

**Runtime Validation Flow:**
> Flow:
> Input Data -> Deserializer -> Validation Framework -> **NotEqualValidator.accept()** -> Mutated ValidationResults

**Schema Generation Flow:**
> Flow:
> System Configuration -> Schema Builder -> **NotEqualValidator.updateSchema()** -> Modified Schema Object -> JSON Schema Exporter

