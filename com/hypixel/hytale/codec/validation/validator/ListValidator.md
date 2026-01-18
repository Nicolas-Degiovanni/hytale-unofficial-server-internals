---
description: Architectural reference for ListValidator
---

# ListValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class ListValidator<T> implements LegacyValidator<List<T>> {
```

## Architecture & Concepts
The ListValidator is a fundamental component within the data validation framework, designed to extend element-wise validation to collections. It embodies the **Composite Design Pattern**, acting as a container that applies a single, specific Validator to each element within a List.

Its primary architectural function is to decouple collection traversal logic from the underlying element validation logic. Instead of requiring developers to write boilerplate loops for every list they need to validate, this class provides a reusable, declarative mechanism. A ListValidator is configured with a Validator for type T, and it transparently handles the iteration and application of that validator to a List of T.

This component is a key building block for constructing complex validation rule sets, especially when dealing with deserialized data models that contain lists of sub-objects.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a higher-level service or factory responsible for building a validation chain. It is always created with a dependency on another Validator instance, which defines the per-element validation rule.
- **Scope:** Short-lived and stateless. Its lifetime is typically scoped to a single, complete validation operation. It holds no state other than its injected dependency.
- **Destruction:** The object is eligible for garbage collection as soon as the `accept` method call completes and its reference goes out of scope. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Effectively Immutable**. The internal `validator` field is private, final (by convention, though not explicitly declared), and set only once via the constructor. The ListValidator instance itself does not mutate its own state during operation. All results are written to the externally provided ValidationResults object.
- **Thread Safety:** Conditionally thread-safe. The ListValidator itself contains no locks and performs no unsafe state modifications. Its safety is therefore entirely dependent on the injected `Validator<T>` instance. If the wrapped validator is thread-safe, multiple threads can safely share a single ListValidator instance.

   **WARNING:** The `accept` method mutates the passed-in ValidationResults parameter. Concurrent calls to `accept` on the *same* ValidationResults instance from multiple threads will lead to race conditions and corrupted results. Each validation task should use a distinct ValidationResults object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(List<T> ts, ValidationResults results) | void | O(N) | Executes the validation logic. Iterates through the input list `ts` and applies the configured validator to each element. The complexity is linear, directly proportional to the number of items in the list. |

## Integration Patterns

### Standard Usage
The primary pattern is to compose validators. First, obtain or create a validator for the element type, then wrap it in a ListValidator to handle collections of that type.

```java
// 1. Define or obtain a validator for the element type (e.g., a String)
Validator<String> stringValidator = new StringLengthValidator(1, 50);

// 2. Construct the ListValidator, injecting the element validator
ListValidator<String> listValidator = new ListValidator<>(stringValidator);

// 3. Prepare the data and results container
List<String> usernames = List.of("admin", "guest", "testuser");
ValidationResults results = new ValidationResults();

// 4. Execute validation
listValidator.accept(usernames, results);

// 5. Check results
if (results.hasErrors()) {
    // Handle validation failures
}
```

### Anti-Patterns (Do NOT do this)
- **Null Dependency:** Do not construct a ListValidator with a null validator. This will not fail at construction but will cause a `NullPointerException` at runtime when `accept` is called. The framework expects a valid, non-null dependency.
- **Stateful Element Validators:** Avoid injecting element validators that maintain internal state across `accept` calls. This can lead to unpredictable behavior and race conditions if the ListValidator is ever reused. Validators should be stateless.

## Data Pipeline
The ListValidator acts as a processing stage in a larger data validation pipeline. It does not source or sink data itself but transforms an input list into a set of validation results by applying a rule.

> Flow:
> `List<T>` (Input Data) -> **ListValidator** (Applies inner `Validator<T>` to each element) -> `ValidationResults` (Aggregated Output)

