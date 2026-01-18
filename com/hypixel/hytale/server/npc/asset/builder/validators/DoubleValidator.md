---
description: Architectural reference for DoubleValidator
---

# DoubleValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Framework Component

## Definition
```java
// Signature
public abstract class DoubleValidator extends Validator {
```

## Architecture & Concepts
The DoubleValidator is an abstract base class that forms a foundational component of the server-side NPC asset validation pipeline. Its primary role is to establish a formal contract for validating double-precision floating-point values found within NPC definition files.

This class acts as a strategy pattern implementation for data integrity. Instead of embedding validation logic directly within the asset parsing code, the system delegates validation to concrete implementations of DoubleValidator. This decouples the parsing mechanism from the business rules, allowing for modular, reusable, and easily testable validation logic.

The static utility method, compare, centralizes all relational logic, ensuring consistent floating-point comparisons across the entire validation framework and preventing common pitfalls associated with direct equality checks on doubles.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of DoubleValidator are instantiated by the NPC asset loading system, specifically by the asset builder or factory responsible for processing raw asset data. An instance is created when a rule requiring double validation is encountered. The abstract class itself is never directly instantiated.
- **Scope:** The lifetime of a DoubleValidator instance is extremely short and transient. It exists only for the duration of a single validation operation on a single data field during the asset loading phase.
- **Destruction:** The object is eligible for garbage collection immediately after its test method has been invoked and the result has been processed by the asset loader. There are no explicit destruction or cleanup requirements.

## Internal State & Concurrency
- **State:** This abstract class is entirely stateless. It defines a contract and provides a pure static utility method. Concrete implementations are strongly expected to be stateless as well, operating solely on the parameters passed to their methods.
- **Thread Safety:** The class is inherently thread-safe. The static compare method is a pure function with no side effects. As concrete implementations are expected to be stateless, they are safe for concurrent use by a multi-threaded asset loading system without requiring any external locking or synchronization.

**WARNING:** Introducing mutable state into a concrete subclass of DoubleValidator is a severe design violation and will lead to race conditions and non-deterministic behavior if the asset pipeline is multi-threaded.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(double) | abstract boolean | O(1) | Executes the core validation logic. Must be implemented by subclasses. |
| compare(double, RelationalOperator, double) | static boolean | O(1) | A safe, centralized utility for performing relational comparisons on doubles. |
| errorMessage(double) | abstract String | O(1) | Generates a context-free error message. Must be implemented by subclasses. |
| errorMessage(double, String) | abstract String | O(1) | Generates a context-aware error message, typically including the property name. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is to extend this class to create a specific, named validation rule. The asset system then discovers and applies this rule during asset loading.

```java
// Example: A concrete validator for NPC health values
public class HealthValidator extends DoubleValidator {
    @Override
    public boolean test(double health) {
        // Health must be greater than 0
        return DoubleValidator.compare(health, RelationalOperator.Greater, 0.0);
    }

    @Override
    public String errorMessage(double health) {
        return "Health value " + health + " must be positive.";
    }

    @Override
    public String errorMessage(double health, String propertyName) {
        return "Property '" + propertyName + "' has invalid health value " + health + ". It must be positive.";
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Implementing Logic in Parsers:** Do not bypass the validator framework by placing validation logic directly inside asset parsing code. This creates tight coupling and makes rules difficult to manage.
- **Stateful Implementations:** Do not add fields or any other mutable state to a subclass. Validators must be reusable and thread-safe.
- **Ignoring Error Messages:** Do not implement the test method without providing meaningful, corresponding implementations for the errorMessage methods. This hinders debugging of invalid asset files.

## Data Pipeline
The DoubleValidator is a critical gatekeeper in the data flow from raw asset files to in-memory game objects.

> Flow:
> NPC Asset File (e.g., JSON) -> HOCON Parser -> Asset Builder -> **DoubleValidator Subclass** -> Validation Result (Pass/Fail) -> NPC Object Instantiation or Error Log<ctrl63>

