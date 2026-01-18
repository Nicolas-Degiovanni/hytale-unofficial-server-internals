---
description: Architectural reference for MapValueValidator
---

# MapValueValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Component / Strategy

## Definition
```java
// Signature
public class MapValueValidator<V> implements Validator<Map<?, V>> {
```

## Architecture & Concepts
The MapValueValidator is a structural component within the Hytale Codec validation framework. It does not perform validation on primitive data itself; instead, it implements the **Strategy Pattern** to delegate the validation of each *value* within a Map to a separate, specialized Validator instance.

Its primary role is to traverse a Map collection and apply a consistent set of validation rules to every value element, ignoring the keys. This design decouples the logic of collection iteration from the logic of element-level validation, promoting composition and reusability. For instance, a single IntegerValidator can be composed within a MapValueValidator to validate a map of settings, or within a ListValidator to validate a list of scores.

This class is a fundamental building block for constructing complex validation graphs for configuration files, network payloads, or any data structure represented as a nested Map.

## Lifecycle & Ownership
- **Creation:** An instance is created programmatically, typically by a builder or factory responsible for assembling a complete validation chain. It is configured at instantiation by providing the delegate Validator via its constructor.
- **Scope:** The lifetime of a MapValueValidator is bound to the parent object that created and holds it, such as a root Validator or a temporary validation context. It is designed to be a transient object.
- **Destruction:** The object is eligible for garbage collection as soon as its parent validator goes out of scope. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state is minimal and effectively immutable. It consists of a single private field, a reference to the delegate Validator for the map values. This reference is set during construction and cannot be changed thereafter.
- **Thread Safety:** This class is conditionally thread-safe. Its own logic is stateless, but its overall thread safety is entirely dependent on the thread safety of the delegate Validator provided during construction. If the delegate is thread-safe, the MapValueValidator can be safely used across multiple threads.

**Warning:** The validation framework does not enforce thread safety on Validator implementations. Assume all validators are confined to a single thread unless explicitly documented otherwise. Sharing a validation chain containing a stateful, non-thread-safe delegate across threads will lead to race conditions and unpredictable behavior.

## API Surface
The public contract focuses on executing validation and contributing to schema generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(Map, ValidationResults) | void | O(N * C) | Executes the validation logic. Iterates through all N values in the input map and invokes the delegate validator for each, where C is the complexity of the delegate. |
| updateSchema(SchemaContext, Schema) | void | O(P) | Recursively updates a target schema. Traverses all P properties in the schema and applies the delegate validator's schema updates. Throws IllegalArgumentException if the target is not an ObjectSchema. |
| getValueValidator() | Validator | O(1) | Returns the configured delegate validator. |

## Integration Patterns

### Standard Usage
The primary pattern is composition, where a MapValueValidator is configured with a specific validator to enforce rules on the values of a map.

```java
// Assume the existence of a pre-configured validator for positive integers.
Validator<Integer> positiveIntegerValidator = new PositiveIntegerValidator();

// Create the map validator, delegating all value validation to the integer validator.
Validator<Map<String, Integer>> settingsValidator = new MapValueValidator<>(positiveIntegerValidator);

// Prepare data and results context
ValidationResults results = new ValidationResults();
Map<String, Integer> gameSettings = Map.of("playerHealth", 100, "maxPartySize", 4, "initialMana", -10);

// Execute validation
settingsValidator.accept(gameSettings, results);

// The 'results' object will now contain a validation error for the 'initialMana' entry.
```

### Anti-Patterns (Do NOT do this)
- **Null Delegation:** Do not construct a MapValueValidator with a null delegate validator. This will result in a NullPointerException during the call to accept or updateSchema. The constructor should be treated as if it requires a non-null argument.
- **Schema Mismatch:** Calling updateSchema with a target Schema that is not an ObjectSchema will cause a runtime IllegalArgumentException. The caller is responsible for ensuring the validator's structure aligns with the schema's structure.

## Data Pipeline
The MapValueValidator acts as a dispatcher in the data validation flow. It receives a collection and routes each individual value to a sub-processor.

> Flow:
> Parent Validator -> **MapValueValidator.accept()** -> (For each value in Map) -> Delegate Validator.accept() -> ValidationResults

