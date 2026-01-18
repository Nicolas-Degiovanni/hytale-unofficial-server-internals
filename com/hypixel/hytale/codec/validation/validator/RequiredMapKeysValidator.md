---
description: Architectural reference for RequiredMapKeysValidator
---

# RequiredMapKeysValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient Strategy Object

## Definition
```java
// Signature
public class RequiredMapKeysValidator<T> implements Validator<Map<T, ?>> {
```

## Architecture & Concepts
The RequiredMapKeysValidator is a concrete implementation of the Validator strategy pattern, designed to enforce structural integrity on Map-like data objects within the Hytale Codec framework. Its primary responsibility is to guarantee that a given Map contains a predefined set of keys.

This component serves a critical dual purpose within the engine's data serialization and validation pipeline:

1.  **Runtime Validation:** Through its implementation of the `accept` method, it performs a direct, runtime check against a Map instance. If any of the configured keys are missing, it reports a failure to the provided ValidationResults collector, effectively halting or flagging the data processing operation.

2.  **Static Schema Enrichment:** The `updateSchema` method allows this validator to contribute to the static definition of the data structure it validates. It injects the list of required keys into the target ObjectSchema, typically as an enumeration. This allows upstream systems, such as development tools, editors, or static analysis engines, to understand the data contract without needing to instantiate and inspect a live object.

Architecturally, it acts as a self-contained rule that bridges a declarative configuration (the list of required keys) with both the dynamic validation engine and the static schema repository.

### Lifecycle & Ownership
- **Creation:** Instances are not intended for direct creation by end-users. They are typically instantiated by a higher-level schema builder or a declarative configuration loader (e.g., parsing a JSON or HOCON file that defines validation rules). The constructor is supplied with the array of keys that this specific validator instance will enforce.
- **Scope:** The lifetime of a RequiredMapKeysValidator is tied to the Schema or validation ruleset to which it belongs. It is a lightweight, stateful object whose state is defined entirely at creation. It persists as long as its parent Schema is in memory.
- **Destruction:** Cleanup is managed by the Java Garbage Collector. There are no native resources or explicit destruction protocols associated with this class.

## Internal State & Concurrency
- **State:** The validator's state consists of a single field: the `array` of required keys. This state is provided at construction and is **immutable** for the lifetime of the object. The validator does not mutate its own state during validation operations.
- **Thread Safety:** This class is inherently thread-safe. Its internal state is read-only after construction. The `accept` and `updateSchema` methods are pure functions with respect to the validator's state, operating exclusively on their method arguments. A single instance can be safely shared and used by multiple threads to validate different Map objects concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RequiredMapKeysValidator(T[] array) | constructor | O(1) | Creates a new validator instance configured to require the provided keys. |
| accept(Map, ValidationResults) | void | O(N) | Executes the validation logic. Iterates through N required keys and checks for their presence in the input map. |
| updateSchema(SchemaContext, Schema) | void | O(N) | Modifies the target Schema to include the N required keys, typically by setting the `propertyNames` field. Throws ClassCastException if the target is not an ObjectSchema. |

## Integration Patterns

### Standard Usage
This validator is not meant to be invoked directly. It is configured as part of a larger schema definition, which is then used by a validation service.

```java
// Hypothetical schema builder demonstrating conceptual usage
ObjectSchema playerConfigSchema = new ObjectSchema();

// The validator is created and attached to a schema rule
playerConfigSchema.addValidator(
    new RequiredMapKeysValidator<>(new String[]{"name", "uuid", "last_seen"})
);

// A central service would later use this schema to validate data
ValidationResults results = ValidationService.validate(playerDataMap, playerConfigSchema);
if (results.hasFailed()) {
    // Handle invalid player configuration
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not manually instantiate and call `accept` on this validator for one-off checks. The Hytale Codec system is designed for centralized, schema-driven validation. Bypassing the schema registry undermines the system's integrity and static analysis capabilities.
- **Incorrect Schema Target:** Passing a Schema that is not an `ObjectSchema` (or a subclass) to the `updateSchema` method will cause a runtime `ClassCastException`. The framework expects validators to be paired with compatible schema types.

## Data Pipeline
The RequiredMapKeysValidator participates in two distinct data flows depending on the engine's operational phase.

**1. Runtime Validation Flow:**
> Flow:
> Raw Data (e.g., from Network or Disk) -> Deserializer -> `Map<T, ?>` Instance -> Validation Engine -> **RequiredMapKeysValidator.accept()** -> ValidationResults

**2. Schema Build-Time Flow:**
> Flow:
> Configuration Source (e.g., HOCON file) -> Schema Parser -> Schema Builder -> **RequiredMapKeysValidator.updateSchema()** -> Finalized In-Memory ObjectSchema

