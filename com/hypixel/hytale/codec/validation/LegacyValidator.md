---
description: Architectural reference for LegacyValidator
---

# LegacyValidator

**Package:** com.hypixel.hytale.codec.validation
**Type:** Interface / Contract

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public interface LegacyValidator<T> extends Validator<T> {
```

## Architecture & Concepts
The LegacyValidator interface is a transitional component within the data codec and schema validation framework. It serves as a compatibility layer, allowing older validation logic to coexist with the modern schema system defined by the primary Validator interface.

**CRITICAL WARNING:** This interface is marked as deprecated and slated for removal. Its existence signifies technical debt. All implementations of LegacyValidator are considered high-priority refactoring targets. The core architectural purpose of this interface is to isolate and flag outdated validation patterns, preventing them from propagating further into the engine. It acts as a formal contract for components that have not yet been migrated to the current validation and schema evolution pipeline.

## Lifecycle & Ownership
As an interface, LegacyValidator itself does not have a lifecycle. However, the concrete classes that implement it do.

- **Creation:** Implementations are typically instantiated by a central ValidatorFactory or a dependency injection container when processing a data schema that references a legacy validation rule.
- **Scope:** The lifetime of a LegacyValidator implementation is generally transient, scoped to a single validation operation. It is created, used to validate an object, and then discarded.
- **Destruction:** Garbage collected after the validation process completes. There are no explicit teardown or resource release requirements defined by this contract.

## Internal State & Concurrency
- **State:** The contract is stateless. However, implementations may be stateful. It is strongly recommended that any implementation be immutable or that its state be confined to the scope of a single `accept` method call.
- **Thread Safety:** The contract does not enforce thread safety. Implementations that are shared across validation contexts **must be thread-safe**. Given that the validation pipeline may operate in parallel, assuming a single-threaded context is unsafe. The default implementation of `updateSchema` is not thread-safe due to its use of System.err.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, ValidationResults) | void | Varies | **[Legacy]** The primary validation entry point. Applies validation rules to the input object and reports failures to the ValidationResults collector. |
| updateSchema(SchemaContext, Schema) | void | O(1) | **[Warning]** Inherited default method. The implementation is a no-op that prints an error to standard error, indicating that legacy validators do not support dynamic schema evolution. |

## Integration Patterns

### Standard Usage
The standard pattern is **refactoring**, not implementation. New code must never implement this interface. Existing code should be migrated away from it.

The following example shows how an older part of the system might invoke a legacy validator, a pattern that should be actively sought out and removed.

```java
// This pattern is DEPRECATED and should be removed.
void validateLegacyObject(MyObject obj, ValidatorRegistry registry) {
    // The registry might return a LegacyValidator implementation
    Validator<MyObject> validator = registry.getValidatorFor(MyObject.class);
    ValidationResults results = new ValidationResults();

    // The validation system invokes the contract
    validator.accept(obj, results);

    if (results.hasErrors()) {
        // Handle validation failure
    }
}
```

### Anti-Patterns (Do NOT do this)
- **New Implementations:** Under no circumstances should a new class be created that implements LegacyValidator. The correct approach is to implement the modern Validator interface directly.
- **Ignoring Deprecation:** Continuing to use classes that implement this interface without a plan for migration introduces significant risk, as the interface and its support will be removed.
- **Relying on updateSchema:** Calling the `updateSchema` method on a LegacyValidator instance is a critical error. The default implementation does nothing but log to the console, which can mask serious schema configuration problems at runtime.

## Data Pipeline
LegacyValidator is a processing stage within the broader data deserialization and validation pipeline. It represents a fork in the pipeline for handling outdated logic.

> Flow:
> Serialized Data -> Deserializer -> Object Model (`T`) -> **LegacyValidator Implementation** -> ValidationResults -> Application Logic

