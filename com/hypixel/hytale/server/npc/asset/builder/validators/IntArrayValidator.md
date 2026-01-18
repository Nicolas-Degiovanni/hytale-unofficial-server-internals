---
description: Architectural reference for IntArrayValidator
---

# IntArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class (Template)

## Definition
```java
// Signature
public abstract class IntArrayValidator extends Validator {
```

## Architecture & Concepts
The IntArrayValidator class is an abstract base class that establishes a formal contract for all validation logic operating on integer arrays. It is a foundational component within the server's asset processing and configuration pipeline. Its primary role is not to perform validation itself, but to define the required interface that concrete validation strategies must implement.

This design employs the **Strategy Pattern**, where each subclass of IntArrayValidator represents a specific, interchangeable validation algorithm (e.g., checking array length, value ranges, or uniqueness). A higher-level system, such as an asset builder, can be configured with a collection of these validator strategies to enforce data integrity rules without being tightly coupled to the specific validation logic.

By extending the base Validator class, it participates in a broader, polymorphic validation framework that can handle various data types, not just integer arrays.

### Lifecycle & Ownership
- **Creation:** This abstract class is never instantiated directly. Concrete subclasses are instantiated by factory methods or dependency injection within the asset building framework when a specific validation rule is required for a given asset property.
- **Scope:** Instances of concrete subclasses are extremely short-lived (transient). They are typically created, used for a single validation check, and then immediately become eligible for garbage collection. They hold no state beyond the duration of a single method call.
- **Destruction:** Managed entirely by the Java Garbage Collector. Due to their transient nature, they impose a negligible memory footprint.

## Internal State & Concurrency
- **State:** This class is fundamentally **stateless**. All methods operate exclusively on the parameters provided at invocation time. Subclasses are strictly expected to adhere to this stateless design.
- **Thread Safety:** The class is inherently **thread-safe**. As it contains no mutable state, a single instance of a concrete subclass can be safely shared and executed by multiple threads simultaneously without locks or synchronization. This is a critical feature for enabling parallel processing of game assets.

## API Surface
The public contract is entirely abstract, forcing subclasses to provide concrete implementations for the validation and error reporting logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int[] var1) | abstract boolean | O(N) | The core validation logic. Must be implemented to evaluate the provided integer array. Complexity is dependent on the subclass implementation. |
| errorMessage(int[] var1, String var2) | abstract String | O(1) | Generates a context-rich error message. The String parameter typically provides the name of the field being validated. |
| errorMessage(int[] var1) | abstract String | O(1) | Generates a generic error message when no additional context is available. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke this class directly. Instead, they create a concrete implementation that encapsulates a specific business rule. The framework then uses this implementation.

**Example: Implementing a validator to ensure an array has exactly 3 elements.**
```java
// 1. Create a concrete implementation
public class ArrayLengthValidator extends IntArrayValidator {
    private static final int REQUIRED_LENGTH = 3;

    @Override
    public boolean test(int[] array) {
        return array != null && array.length == REQUIRED_LENGTH;
    }

    @Override
    public String errorMessage(int[] array, String fieldName) {
        return String.format(
            "Validation failed for field '%s': Expected array length of %d, but got %d.",
            fieldName,
            REQUIRED_LENGTH,
            array == null ? 0 : array.length
        );
    }

    @Override
    public String errorMessage(int[] array) {
        return errorMessage(array, "[unspecified field]");
    }
}

// 2. The asset pipeline would then use it internally
// (This code would exist in a separate AssetBuilder class)
IntArrayValidator lengthValidator = new ArrayLengthValidator();
int[] dataToValidate = {10, 20}; // Invalid data

if (!lengthValidator.test(dataToValidate)) {
    String error = lengthValidator.errorMessage(dataToValidate, "position");
    // Log or throw an AssetLoadException with the error message
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create an instance via `new IntArrayValidator()` will result in a compile-time error. You must subclass it.
- **Stateful Implementations:** Subclasses must not contain mutable instance variables. Doing so breaks thread safety and violates the design contract, leading to unpredictable behavior in a multi-threaded asset loading environment.
- **Complex Logic in Constructors:** All validation logic belongs in the `test` method. Constructors should be lightweight and dependency-free.

## Data Pipeline
This component acts as a gatekeeper within a larger data deserialization and validation pipeline. It does not transform data but rather asserts its correctness.

> Flow:
> Raw Asset File (e.g., JSON) -> Deserializer -> Raw `int[]` -> **Concrete IntArrayValidator** -> Validation Result (Pass/Fail) -> Asset Instantiation or Error Reporting

