---
description: Architectural reference for IntArrayValidator
---

# IntArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class IntArrayValidator implements Validator<int[]> {
```

## Architecture & Concepts
The IntArrayValidator is a specialized component within the Hytale codec and schema validation framework. It embodies the **Composite** design pattern, acting as a container that applies a single validation rule to each element of a primitive integer array.

Its primary role is to bridge the gap between array-level validation and element-level validation. Instead of implementing array-specific logic itself, it delegates the validation of each integer to a wrapped, more granular Validator instance. This design promotes reusability and separation of concerns; for example, the same Integer validator that checks for a valid range can be used for a single field or, via IntArrayValidator, for every element in an array.

This class is a crucial part of the system that dynamically constructs validation pipelines based on a formal Schema definition. When the schema specifies an array of integers, this validator is instantiated to enforce the schema's constraints on that array's contents.

### Lifecycle & Ownership
- **Creation:** IntArrayValidator is not intended for direct instantiation by end-users. It is constructed by a higher-level factory or builder, such as a Schema-to-Validator compiler, during the setup of a validation graph. The factory provides the required `Validator<Integer>` dependency during construction.
- **Scope:** The lifetime of an IntArrayValidator instance is typically ephemeral, scoped to a single, complete validation operation. It does not persist between distinct validation tasks.
- **Destruction:** The object is eligible for garbage collection as soon as the validation process it was part of has completed and no external references remain.

## Internal State & Concurrency
- **State:** The class is effectively immutable after construction. Its only state is the final `validator` field, which holds the reference to the delegate validator for individual integers. It does not cache results or modify its internal configuration during its lifecycle.
- **Thread Safety:** This class contains no internal locks or mutable state, making it conditionally thread-safe. Its safety in a concurrent environment is entirely dependent on the thread safety of the injected `Validator<Integer>` instance and the `ValidationResults` object passed to the `accept` method. If the delegate and the results collector are thread-safe, this validator can be safely used across multiple threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int[], ValidationResults) | void | O(N) | Iterates through the input array, applying the wrapped validator to each element. N is the number of elements in the array. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Delegates schema modification to the wrapped validator. Throws IllegalArgumentException if the target Schema is not an ArraySchema. |

## Integration Patterns

### Standard Usage
This validator is used implicitly by the codec framework. A developer defines a schema, and the framework selects and configures this validator to enforce it.

```java
// Conceptual example of framework-level usage

// 1. A factory creates the necessary validators based on a schema
Validator<Integer> intRangeValidator = new IntRangeValidator(0, 255);
IntArrayValidator intArrayValidator = new IntArrayValidator(intRangeValidator);

// 2. During data processing, the validator is invoked
int[] pixelData = {10, 50, 255, 300}; // Contains an invalid value
ValidationResults results = new ValidationResults();
intArrayValidator.accept(pixelData, results);

// 3. Results now contain a validation error for the value 300
boolean isValid = results.isEmpty(); // false
```

### Anti-Patterns (Do NOT do this)
- **Schema Mismatch:** Calling `updateSchema` with a Schema instance that is not an `ArraySchema` will result in a runtime `IllegalArgumentException`. The validator is tightly coupled to the structure it is designed to validate.
- **Null Delegate:** Constructing an `IntArrayValidator` with a null delegate validator will cause a `NullPointerException` when `accept` or `updateSchema` is called. The framework is expected to prevent this.

## Data Pipeline
The IntArrayValidator functions as a processing stage within a larger data validation pipeline, typically during deserialization or data ingestion.

> Flow:
> Raw Data -> Deserializer -> `int[]` Payload -> **IntArrayValidator** -> (Loop over each element) -> Delegate `Validator<Integer>` -> Aggregated `ValidationResults`

