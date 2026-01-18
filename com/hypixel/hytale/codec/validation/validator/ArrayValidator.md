---
description: Architectural reference for ArrayValidator
---

# ArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class ArrayValidator<T> implements Validator<T[]> {
```

## Architecture & Concepts
The ArrayValidator is a fundamental component of the Hytale Codec validation framework. It embodies the *Composite* design pattern, enabling the application of a single validation rule to each element within an array. It does not contain any validation logic itself; instead, it delegates the validation of each item to an inner, composed Validator instance.

This class acts as a structural validator. Its primary responsibility is to traverse an array data structure and ensure that a specific item-level validator is executed for every element. This makes it a critical building block for constructing validation graphs for complex configuration objects and network payloads that contain collections.

Furthermore, its integration with the schema system via the `updateSchema` method allows it to dynamically contribute to the generation of a formal data schema. It enforces that the target schema is an `ArraySchema` and recursively populates the item-level schema definition using its composed validator.

## Lifecycle & Ownership
- **Creation:** An ArrayValidator is typically instantiated by a higher-level factory or builder responsible for assembling a complete validation chain. It is created with a pre-existing, non-null Validator instance that defines the per-element validation logic. It is not a managed service or singleton.
- **Scope:** The lifetime of an ArrayValidator is tied to the specific validation task for which it was constructed. It is a short-lived, transient object that persists only as long as the root validator object that contains it.
- **Destruction:** The object is eligible for garbage collection as soon as the validation process completes and all references to the validation graph are released. It requires no explicit cleanup or destruction logic.

## Internal State & Concurrency
- **State:** The internal state is minimal and immutable after construction, consisting of a single reference to the composed `validator`. The ArrayValidator itself is stateless with respect to the data it processes.
- **Thread Safety:** This class is **conditionally thread-safe**. An ArrayValidator instance can be safely shared and reused across multiple threads if, and only if, the composed inner `validator` is also thread-safe. The primary concurrency hazard is the mutable `ValidationResults` object passed to the `accept` method. This results object must **not** be shared between threads performing concurrent validations without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ArrayValidator(Validator<T>) | constructor | O(1) | Constructs the validator, composing an inner validator for the array elements. |
| accept(T[], ValidationResults) | void | O(N) | Executes the validation. Iterates through the input array and applies the inner validator to each element. N is the number of elements in the array. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Updates the provided schema definition. Throws IllegalArgumentException if the target is not an ArraySchema. |
| getValidator() | Validator<T> | O(1) | Returns the composed inner validator. |

## Integration Patterns

### Standard Usage
The ArrayValidator is used to build complex validation rules by composition. It is rarely used in isolation but rather as part of a larger validation graph.

```java
// Example: Validate an array of strings, ensuring none are null or empty.
Validator<String> elementValidator = new NotEmptyValidator();
Validator<String[]> listValidator = new ArrayValidator<>(elementValidator);

ValidationResults results = new ValidationResults();
String[] data = new String[]{"first", "", "third"};

// The listValidator will iterate and use elementValidator on each item.
listValidator.accept(data, results);

// The 'results' object will now contain an error for the second element.
if (results.hasErrors()) {
    // Handle validation failure...
}
```

### Anti-Patterns (Do NOT do this)
- **Null Composition:** Do not construct an ArrayValidator with a null inner validator. This will lead to a `NullPointerException` during the `accept` or `updateSchema` calls, as the class performs no internal null checks on its composed dependency.
- **Schema Mismatch:** Do not call `updateSchema` with a `Schema` object that is not an instance of `ArraySchema`. This violates the component's contract and will raise an `IllegalArgumentException`, halting the schema generation process.
- **Stateful Inner Validators:** Avoid composing an ArrayValidator with an inner validator that is stateful and not thread-safe if the ArrayValidator instance will be shared across threads. This can lead to unpredictable validation results and race conditions.

## Data Pipeline
The ArrayValidator acts as a dispatcher in the data validation pipeline. It deconstructs an array and forwards each element down the chain to its composed sub-validator.

> Flow:
> Input `T[]` -> **ArrayValidator** Iteration -> (For each element `T`) -> Inner `Validator<T>` -> `ValidationResults` Mutation

