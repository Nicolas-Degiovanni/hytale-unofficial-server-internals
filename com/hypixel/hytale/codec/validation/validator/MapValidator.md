---
description: Architectural reference for MapValidator
---

# MapValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class MapValidator<K, V> implements Validator<Map<K, V>> {
```

## Architecture & Concepts
The MapValidator is a composite component within the Hytale Codec validation framework. It does not implement any validation logic itself; instead, it acts as a structural validator that delegates the validation of a map's keys and values to two other, specialized Validator instances. This compositional design allows for the construction of complex and reusable validation rules for nested data structures.

Its primary role is to traverse a Map data structure and apply a specific Validator to every key and another to every value, aggregating the results. This makes it a fundamental building block for validating configuration objects, network payloads, or any key-value data that must adhere to a strict contract.

A secondary but critical function is its role in schema modification. The `updateSchema` method translates the validation rules for keys and values into formal constraints within an ObjectSchema, effectively building a machine-readable definition of the valid data structure.

## Lifecycle & Ownership
- **Creation:** A MapValidator is instantiated dynamically, typically by a higher-level factory or builder responsible for assembling a complete validation graph. It is constructed by providing concrete Validator implementations for its keys and values. It is not a managed service or singleton.
- **Scope:** The lifetime of a MapValidator is ephemeral. It is scoped to the lifecycle of the parent Validator that contains it and typically exists only for the duration of a single, complete validation or schema generation operation.
- **Destruction:** The object is eligible for garbage collection as soon as the root validation process is complete and all references to it are dropped. It requires no explicit destruction or resource cleanup.

## Internal State & Concurrency
- **State:** The internal state consists of two final references to the key and value validators. This state is immutable after the object is constructed. The MapValidator does not cache results or modify its own state during operation.
- **Thread Safety:** This class is inherently thread-safe, contingent upon the thread safety of the composed key and value validators. As validators within the codec system are designed to be stateless, this component can be safely used across multiple threads without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(Map, ValidationResults) | void | O(N) | Iterates the map, applying the key and value validators to each entry. N is the number of entries in the map. |
| updateSchema(SchemaContext, Schema) | void | O(P) | Modifies the target ObjectSchema to reflect the key/value validation rules. P is the number of properties in the schema. Throws IllegalArgumentException if the target is not an ObjectSchema. |

## Integration Patterns

### Standard Usage
The MapValidator is intended to be composed with other validators to build a comprehensive validation rule set.

```java
// 1. Define validators for the map's keys and values
Validator<String> keyValidator = new StringValidator().pattern("^[a-z_]+$");
Validator<Integer> valueValidator = new IntegerValidator().min(0);

// 2. Compose them into a MapValidator instance
MapValidator<String, Integer> settingsValidator = new MapValidator<>(keyValidator, valueValidator);

// 3. Use the composite validator to check a map instance
ValidationResults results = new ValidationResults();
Map<String, Integer> userSettings = Map.of("volume", 100, "fov", 90);
settingsValidator.accept(userSettings, results);

if (results.hasErrors()) {
    // Handle validation failures
}
```

### Anti-Patterns (Do NOT do this)
- **Mismatched Schema Target:** Calling `updateSchema` with a Schema type that is not an ObjectSchema will cause a runtime crash. The validator is tightly coupled to the structure of an ObjectSchema.
- **Null Validator Injection:** The constructor does not perform null checks on its arguments. Constructing a MapValidator with a null key or value validator will result in a NullPointerException during execution of `accept` or `updateSchema`.

## Data Pipeline
The MapValidator acts as a dispatcher in two distinct data flows: validation and schema generation.

**Validation Flow:**
> Map Instance -> **MapValidator.accept()** -> (For each entry) -> Key Validator & Value Validator -> Aggregated ValidationResults

**Schema Generation Flow:**
> ObjectSchema -> **MapValidator.updateSchema()** -> Key Validator & Value Validator -> Modified ObjectSchema with new constraints

