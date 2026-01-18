---
description: Architectural reference for SequentialDoubleArrayValidator
---

# SequentialDoubleArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Utility

## Definition
```java
// Signature
public class SequentialDoubleArrayValidator implements Validator<double[]> {
```

## Architecture & Concepts
The SequentialDoubleArrayValidator is a stateless, single-purpose component within the Hytale Codec validation framework. Its sole function is to enforce a data integrity constraint on arrays of doubles, ensuring that all elements are arranged in a monotonically increasing sequence.

This validator acts as a pluggable rule that can be attached to a schema definition. It is primarily used to validate ordered data structures such as animation keyframe timelines, function graphs, or configuration values that must strictly increase.

The component's behavior is controlled by a single boolean flag, `allowEquals`, which toggles between two modes:
1.  **Strictly Increasing:** `[1.0, 2.0, 3.0]` is valid, but `[1.0, 2.0, 2.0]` is not.
2.  **Non-Decreasing:** `[1.0, 2.0, 2.0]` is valid.

Two pre-configured static instances, `NEQ_INSTANCE` (strictly increasing) and `ALLOW_EQ_INSTANCE` (non-decreasing), are provided to serve as canonical, reusable singletons for the most common use cases.

## Lifecycle & Ownership
-   **Creation:** The primary method of access is through the static final instances `NEQ_INSTANCE` and `ALLOW_EQ_INSTANCE`. Direct instantiation via `new SequentialDoubleArrayValidator(boolean)` is supported but should only be used by higher-level schema builders or configuration systems.
-   **Scope:** As a stateless utility, an instance can be considered a singleton. The static instances persist for the entire application lifetime. Dynamically created instances are short-lived and owned by the schema configuration that created them.
-   **Destruction:** Instances are eligible for garbage collection when the owning schema is unloaded. The static instances are never collected.

## Internal State & Concurrency
-   **State:** Immutable. The `allowEquals` field is final and set at construction time. The validator holds no state related to any specific validation operation; all required data is passed as arguments to the `accept` method.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature and lack of side-effects (other than mutating the passed-in `ValidationResults` object) allow a single instance to be safely shared and used across multiple threads concurrently without locks or synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| SequentialDoubleArrayValidator(boolean) | constructor | O(1) | Creates a new validator with the specified equality rule. |
| accept(double[], ValidationResults) | void | O(N) | Executes the validation logic. Iterates the array once, failing if a non-sequential value is found. |
| updateSchema(SchemaContext, Schema) | void | O(1) | No-op. This validator does not contribute any metadata to the schema definition itself. |

## Integration Patterns

### Standard Usage
The validator is invoked by a parent validation system, which passes the target data and a results collector. For ad-hoc validation, it can be called directly.

```java
// Example of direct invocation
ValidationResults results = new ValidationResults();
double[] validData = { 10.5, 20.0, 35.1 };
double[] invalidData = { 10.5, 50.0, 45.0 };

// Use the canonical static instance for strict validation
SequentialDoubleArrayValidator validator = SequentialDoubleArrayValidator.NEQ_INSTANCE;

validator.accept(validData, results); // results.hasFailed() is false
validator.accept(invalidData, results); // results.hasFailed() is true
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Avoid creating new instances when the provided static singletons suffice. This is wasteful and bypasses the intended singleton pattern.
    ```java
    // AVOID
    Validator<double[]> v = new SequentialDoubleArrayValidator(false);

    // PREFER
    Validator<double[]> v = SequentialDoubleArrayValidator.NEQ_INSTANCE;
    ```
-   **Ignoring Results Object:** The `accept` method has a `void` return type. The outcome of the validation is recorded *exclusively* by mutating the passed `ValidationResults` object. Calling the method without checking the results object renders the operation useless.

## Data Pipeline
This component acts as a discrete processing node in a larger data validation pipeline. It does not manage the flow of data itself but rather operates on data provided to it.

> Flow:
> `double[]` (from config file, network packet, etc.) -> **SequentialDoubleArrayValidator.accept()** -> `ValidationResults` (mutated with failure message if validation fails)

