---
description: Architectural reference for ArraySizeValidator
---

# ArraySizeValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class ArraySizeValidator<T> implements Validator<T[]> {
```

## Architecture & Concepts
The ArraySizeValidator is a specialized, single-purpose component within the Hytale Codec and Schema validation framework. It enforces a strict, fixed-size constraint on array-like data structures during the deserialization and validation process.

Architecturally, it embodies the Strategy pattern, where it represents one specific validation rule that can be composed with other validators to form a complete validation chain for a complex object.

Its design serves a dual purpose:
1.  **Runtime Validation:** Through the `accept` method, it performs a direct check on a data instance to ensure its size conforms to the configured rule.
2.  **Schema Generation:** Through the `updateSchema` method, it contributes to the formal definition of the data structure by programmatically setting the `minItems` and `maxItems` properties on an ArraySchema. This ensures that any generated schema (e.g., for client-side validation or API documentation) accurately reflects the server-side validation logic.

This validator is a leaf node in the validation system; it is stateless (after construction) and has no dependencies on other services.

### Lifecycle & Ownership
-   **Creation:** ArraySizeValidator instances are not intended to be created directly by application-level code. They are instantiated by a higher-level factory or builder within the Codec framework, typically when parsing validation annotations or a declarative configuration. The required size is injected via the constructor at this time.
-   **Scope:** The lifetime of an ArraySizeValidator is typically scoped to the lifecycle of the schema or validator set it belongs to. It is a lightweight object that may be created on-demand and discarded after the validation process is complete.
-   **Destruction:** Instances are managed by the Java garbage collector and require no explicit cleanup.

## Internal State & Concurrency
-   **State:** The object's state is **immutable**. The `size` field is final and set only once during construction. The validator holds no reference to the data it validates or the results it produces, making it highly predictable.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state, a single instance can be safely shared and executed by multiple threads concurrently without the need for external locking or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T[] array, ValidationResults results) | void | O(1) | Executes the validation logic. Compares the array's length to the configured size and reports a failure to the ValidationResults object if they do not match. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Modifies a target Schema object to reflect the size constraint. **Warning:** Throws a ClassCastException if the target is not an ArraySchema. |

## Integration Patterns

### Standard Usage
This validator is designed to be used by the validation framework, not directly by service-level code. A builder or factory would compose it into a validation chain.

```java
// Conceptual framework usage
// A schema builder would instantiate and register this validator
// based on a rule like "@Size(16)" on a field.

List<Validator> fieldValidators = new ArrayList<>();
fieldValidators.add(new ArraySizeValidator(16));

// The framework then iterates this list during validation
for (Validator validator : fieldValidators) {
    validator.accept(dataField, validationResults);
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Avoid creating instances of ArraySizeValidator in your application logic. Define validation rules declaratively and let the framework manage validator lifecycle.
-   **Schema Mismatch:** Do not configure this validator to run against a schema that is not an ArraySchema. Doing so will result in a runtime `ClassCastException` during the schema generation phase. The framework should prevent this, but misconfiguration can lead to this state.

## Data Pipeline
The ArraySizeValidator operates at a specific stage in the data validation pipeline. It does not transform data but rather acts as a gatekeeper that asserts a condition on the data.

> Flow:
> Raw Data (e.g., Network Packet) -> Deserializer -> Validation Engine -> **ArraySizeValidator.accept()** -> ValidationResults -> Business Logic

During schema generation, the flow is different:

> Flow:
> Object Definition / Annotations -> Schema Builder -> **ArraySizeValidator.updateSchema()** -> Finalized ArraySchema Document

