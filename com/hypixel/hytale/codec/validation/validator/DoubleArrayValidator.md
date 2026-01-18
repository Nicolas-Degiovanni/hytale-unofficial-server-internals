---
description: Architectural reference for DoubleArrayValidator
---

# DoubleArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Strategy Object

## Definition
```java
// Signature
public class DoubleArrayValidator implements Validator<double[]> {
```

## Architecture & Concepts
The DoubleArrayValidator is a structural component within the schema validation framework. It embodies the **Composite** design pattern, acting as a container that applies a single validation rule to each element of a collection. Its primary role is not to validate the array itself (e.g., its length or uniqueness of its elements), but to delegate the validation of each individual double primitive to a more specialized, injected validator.

This class serves as an adapter, enabling a `Validator<Double>` to operate on a `double[]`. This design is fundamental to the framework's ability to construct complex validation logic from simple, reusable primitives. By composing validators in this manner, the system can represent and enforce nested schema rules, such as "an array of doubles where each double must be between 0.0 and 1.0".

It is a critical link between collection-level schemas (ArraySchema) and primitive-level validation logic.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level factory or builder, typically during the process of converting a declarative Schema definition into an executable validation graph. It is never created directly by application-level code. For example, a `ValidatorFactory` would create a DoubleArrayValidator when it encounters an ArraySchema whose items are of type double.
-   **Scope:** Transient and short-lived. Its lifetime is bound to the validation graph it belongs to. This graph may be cached for performance, but the validator object itself is considered a stateless processing node.
-   **Destruction:** Eligible for garbage collection as soon as the parent validation graph is no longer referenced. It manages no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class holds a single state field: the injected `validator` of type `Validator<Double>`. This reference is set at construction and is not modified thereafter, making the DoubleArrayValidator's configuration effectively immutable. It performs no internal caching.
-   **Thread Safety:** This class is conditionally thread-safe. It contains no locks or mutable instance variables. Its thread safety is entirely dependent on the thread safety of the injected `Validator<Double>` instance and the `ValidationResults` object passed to its methods. If the injected validator is stateless and the `ValidationResults` implementation is thread-safe (or confined to a single thread), then DoubleArrayValidator can be safely used across multiple threads.

**WARNING:** Sharing a stateful inner validator (e.g., one that counts invocations) across threads will lead to race conditions. The framework is designed around the assumption of stateless, reusable validator components.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(double[] ds, ValidationResults results) | void | O(N) | Iterates through the input array and invokes the injected validator for each element. N is the number of elements in the array. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Propagates schema constraints to the inner validator. Throws IllegalArgumentException if the target is not an ArraySchema. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use but is composed by the framework. The conceptual usage pattern involves wrapping a primitive validator to apply it to an array.

```java
// A validator that checks if a double is positive
Validator<Double> positiveDoubleValidator = new PositiveDoubleValidator();

// Use DoubleArrayValidator to apply this rule to an entire array
Validator<double[]> arrayValidator = new DoubleArrayValidator(positiveDoubleValidator);

// Execute the validation
ValidationResults results = new ValidationResults();
double[] data = { 10.5, 5.0, -2.7, 8.0 };
arrayValidator.accept(data, results);

// The 'results' object will now contain a validation error for the value -2.7
```

### Anti-Patterns (Do NOT do this)
-   **Schema Mismatch:** Calling `updateSchema` with a Schema type that is not an ArraySchema will result in an `IllegalArgumentException`. This indicates a severe logic error in the validator construction phase, as the validator type and schema type are misaligned.
-   **Null Injection:** Constructing a DoubleArrayValidator with a null inner validator will cause a `NullPointerException` during the `accept` or `updateSchema` calls. The constructor does not perform null checks, assuming it is created by a trusted factory.

## Data Pipeline
The DoubleArrayValidator acts as a "fan-out" component in the data validation pipeline. It deconstructs a collection into its constituent elements and forwards each one for individual processing.

> Flow:
> `double[]` -> **DoubleArrayValidator** (Iteration) -> `double` (element) -> Injected `Validator<Double>` -> `ValidationResults` (Accumulation)

