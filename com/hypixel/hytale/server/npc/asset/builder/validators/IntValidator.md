---
description: Architectural reference for IntValidator
---

# IntValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class

## Definition
```java
// Signature
public abstract class IntValidator extends Validator {
```

## Architecture & Concepts
The IntValidator class is an abstract base class that defines the contract for all integer-based validation logic within the server-side NPC asset pipeline. It is a foundational component of a Strategy Pattern implementation, where concrete subclasses provide specific validation algorithms for numeric properties of an NPC asset during its loading and construction phase.

This class is not intended for direct instantiation. Instead, it serves as a blueprint, ensuring that all integer validation routines adhere to a consistent interface. This design decouples the asset parsing system from the specific validation rules, allowing new rules to be added without modifying the core asset loading machinery.

The static utility method, compare, centralizes relational logic, providing a standardized and reusable mechanism for subclasses to perform common comparisons. This prevents code duplication and potential inconsistencies in how relational checks are implemented across different validators.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of IntValidator are instantiated by the NPC asset building framework. This typically occurs when an asset definition file is being parsed and its properties require validation against predefined rules.
- **Scope:** The lifetime of an IntValidator subclass instance is extremely brief and transient. It is created on-demand for a single validation operation and does not persist beyond that check.
- **Destruction:** The object is eligible for garbage collection immediately after its test method has been invoked and the result has been processed by the calling system. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** IntValidator and its intended subclasses are designed to be stateless. They do not hold or cache any data between method invocations. All operations are performed exclusively on the input parameters.
- **Thread Safety:** This class is inherently thread-safe due to its stateless design. The static compare method is a pure function and is also safe for concurrent access. Subclasses should maintain this stateless property to ensure safe execution within a multi-threaded asset loading environment.

**WARNING:** Introducing mutable state into a concrete subclass of IntValidator is a severe anti-pattern and will break the concurrency guarantees of the asset validation system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int) | abstract boolean | O(1) | **Contract.** Subclasses must implement this to perform the core validation logic. Returns true if the integer is valid. |
| compare(int, RelationalOperator, int) | static boolean | O(1) | A utility function to perform a relational comparison between two integers. Provided for convenience. |
| errorMessage(int) | abstract String | O(1) | **Contract.** Subclasses must implement this to provide a generic error message if validation fails. |
| errorMessage(int, String) | abstract String | O(1) | **Contract.** Subclasses must implement this to provide a context-specific error message, typically including the name of the property being validated. |

## Integration Patterns

### Standard Usage
The primary integration pattern is extension. A developer creates a new, concrete class that extends IntValidator and implements the required abstract methods to define a specific business rule.

```java
// Example: A validator to ensure an NPC's health is positive.
public class PositiveHealthValidator extends IntValidator {
    @Override
    public boolean test(int value) {
        // Using the provided static helper for clarity and consistency
        return IntValidator.compare(value, RelationalOperator.Greater, 0);
    }

    @Override
    public String errorMessage(int value) {
        return "Health must be a positive number, but was " + value;
    }

    @Override
    public String errorMessage(int value, String propertyName) {
        return "Property '" + propertyName + "' must be positive, but was " + value;
    }
}

// This validator would then be used by the asset system internally.
// Developer code would not typically invoke this directly.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create an instance via `new IntValidator()` is a compile-time error. You must extend the class.
- **Stateful Implementations:** Adding member variables to a subclass that store state across calls violates the design contract and can lead to severe, difficult-to-diagnose concurrency bugs in the asset loader.
- **Complex Logic in `test`:** The `test` method should encapsulate a single, simple validation rule. Complex, multi-stage validations should be composed of multiple, distinct Validator instances.

## Data Pipeline
IntValidator sits within the data validation stage of the server's asset ingestion pipeline. It acts as a gatekeeper, ensuring numeric data conforms to engine and game design constraints before it is used to construct a final NPC object in memory.

> Flow:
> NPC Asset File (JSON/XML) -> Asset Deserializer -> Property Mapper -> **Concrete IntValidator Subclass** -> Validation Result (Pass/Fail) -> NPC Object Construction

