---
description: Architectural reference for FloatArrayValidator
---

# FloatArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient Component

## Definition
```java
// Signature
public class FloatArrayValidator implements Validator<float[]> {
```

## Architecture & Concepts
The FloatArrayValidator is a composite component within the Hytale Codec validation framework. Its primary function is to apply a single, element-level validation rule across every item in an array of primitive floats. It does not contain any validation logic itself; instead, it acts as a container and iterator that delegates the actual validation of each float to a separate, injected **Validator<Float>** instance.

This design follows the **Strategy** or **Decorator** pattern, promoting separation of concerns. It decouples the logic of iterating over an array from the specific validation rules applied to the array's elements. This allows any **Validator<Float>**—such as a range checker or a specific value checker—to be reused for validating entire arrays without modification.

This class is a fundamental building block for constructing complex validation chains for configuration files, network packets, and other structured data that utilize arrays of floating-point numbers.

### Lifecycle & Ownership
-   **Creation:** An instance of FloatArrayValidator is typically created by a higher-level factory or builder within the Codec system, usually in response to parsing a schema that defines an array of floats. It is **not** a singleton.
-   **Scope:** The lifetime of a FloatArrayValidator instance is tied to the parent object that created it, such as a larger schema validator or a codec instance. It is generally short-lived and scoped to a specific validation task.
-   **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection once the parent validator is no longer in scope.

## Internal State & Concurrency
-   **State:** The class is effectively immutable. Its only internal state is a **final** reference to the injected **Validator<Float>**, which is set at construction and cannot be changed thereafter. It does not cache results or maintain state between validation calls.
-   **Thread Safety:** This class is conditionally thread-safe. The FloatArrayValidator itself contains no mutable state or locks. Its thread safety is entirely dependent on the thread safety of the **Validator<Float>** instance provided during construction. If the injected validator is thread-safe, this class can be safely used across multiple threads.

## API Surface
The public contract is minimal, focusing on the implementation of the **Validator** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FloatArrayValidator(Validator<Float>) | constructor | O(1) | Constructs the validator, injecting the element-level validator dependency. |
| accept(float[], ValidationResults) | void | O(N) | Validates each element in the input array. N is the number of elements in the array. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Updates the target **ArraySchema** by delegating to the item validator. Throws **IllegalArgumentException** if the target is not an **ArraySchema**. |

## Integration Patterns

### Standard Usage
The primary pattern is composition. A specific, element-level validator is created first and then passed to the FloatArrayValidator's constructor to apply that logic to an array.

```java
// 1. Obtain or create a validator for a single float (e.g., a range validator)
Validator<Float> elementValidator = new FloatRangeValidator(0.0f, 1.0f);

// 2. Compose the element validator within the FloatArrayValidator
FloatArrayValidator arrayValidator = new FloatArrayValidator(elementValidator);

// 3. Execute validation on a data array
ValidationResults results = new ValidationResults();
float[] validData = { 0.1f, 0.5f, 0.99f };
arrayValidator.accept(validData, results);

// The 'results' object will contain any validation failures.
if (results.hasErrors()) {
    // Handle validation errors
}
```

### Anti-Patterns (Do NOT do this)
-   **Passing Null Validator:** Never construct a FloatArrayValidator with a null dependency. This will lead to a **NullPointerException** when **accept** or **updateSchema** is called.
-   **Schema Mismatch:** Do not call **updateSchema** with a Schema object that is not an instance of **ArraySchema**. The method performs a runtime type check and will throw an **IllegalArgumentException**, halting the schema generation process.

## Data Pipeline
The FloatArrayValidator acts as a processing stage within two distinct data flows: data validation and schema generation.

**Data Validation Flow:**
> Flow:
> Raw Data (File, Network) -> Deserializer -> `float[]` instance -> **FloatArrayValidator** -> Populated **ValidationResults** object

**Schema Generation Flow:**
> Flow:
> Schema Builder -> **FloatArrayValidator**.updateSchema() -> Updates `items` field of **ArraySchema** -> Completed Schema Definition

