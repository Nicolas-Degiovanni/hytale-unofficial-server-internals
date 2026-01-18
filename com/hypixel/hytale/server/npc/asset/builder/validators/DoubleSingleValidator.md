---
description: Architectural reference for DoubleSingleValidator
---

# DoubleSingleValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class DoubleSingleValidator extends DoubleValidator {
```

## Architecture & Concepts
The DoubleSingleValidator is a low-level, immutable component within the server's NPC asset validation framework. Its primary function is to enforce a single relational constraint (e.g., greater than, less than or equal to) on a double-precision floating-point value.

This class acts as a predicate in the data validation pipeline during asset loading and construction. When the server parses an NPC definition file, builders use validators like this one to ensure that numerical properties such as health, speed, or scale fall within acceptable bounds.

Its design emphasizes performance and reusability through a combination of static factory methods and pre-allocated singleton instances for common validation rules (e.g., checking for positive values). Because instances are immutable, they are inherently thread-safe and can be shared freely across the asset loading system.

### Lifecycle & Ownership
- **Creation:** Instances are exclusively created via public static factory methods, such as `greater0` or `greater(double)`. The constructor is private to enforce this pattern. For frequently used validations, like ensuring a value is greater than zero, the system returns a pre-allocated, static singleton instance to prevent unnecessary object creation.
- **Scope:** Validator objects are typically short-lived and transient. They are created, used to perform one or more checks by a builder, and then become eligible for garbage collection. The cached singleton instances, however, persist for the entire lifetime of the server application.
- **Destruction:** All instances, including the cached singletons, are managed by the Java Garbage Collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** Immutable. The internal state, consisting of the `RelationalOperator` and the threshold `value`, is set once at construction time via `final` fields. An instance of DoubleSingleValidator cannot be modified after it is created.
- **Thread Safety:** This class is inherently thread-safe. Due to its immutable nature, a single instance can be safely used by multiple threads simultaneously without any risk of data corruption or race conditions. This is a critical feature for parallelized asset loading processes on the server.

## API Surface
The public API is designed as a set of factories for creating validator instances and a simple `test` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| greater0() | static DoubleSingleValidator | O(1) | Returns a cached singleton instance that validates if a value is > 0.0. |
| greater(double threshold) | static DoubleSingleValidator | O(1) | Creates and returns a new validator that checks if a value is > the specified threshold. |
| greaterEqual0() | static DoubleSingleValidator | O(1) | Returns a cached singleton instance that validates if a value is >= 0.0. |
| test(double value) | boolean | O(1) | Executes the validation logic against the provided value. Returns true if the condition is met. |
| errorMessage(double value, String name) | String | O(1) | Generates a descriptive, human-readable error message if the validation fails. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by higher-level builder or configuration loading systems. The client code obtains a validator instance and passes it to a validation utility or uses it directly as a predicate.

```java
// Example: Validating an NPC's movement speed during asset construction
double speed = npcConfiguration.getSpeed();
String fieldName = "movementSpeed";

// Obtain a pre-cached validator for a common, performance-critical check
DoubleSingleValidator validator = DoubleSingleValidator.greaterEqual0();

if (!validator.test(speed)) {
    throw new AssetValidationException(validator.errorMessage(speed, fieldName));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is `private`. Attempting to call `new DoubleSingleValidator()` will result in a compile-time error. You must always use the provided static factory methods.
- **Redundant Creation:** Avoid creating new validators for common checks inside loops or frequently called methods. Always prefer the cached singletons where applicable.

   **BAD:**
   ```java
   for (Npc npc : allNpcs) {
       // Creates a new object on every iteration, causing unnecessary GC pressure.
       Validator v = DoubleSingleValidator.greater(0.0);
       v.test(npc.getHealth());
   }
   ```

   **GOOD:**
   ```java
   // Retrieve the singleton once and reuse it.
   Validator v = DoubleSingleValidator.greater0();
   for (Npc npc : allNpcs) {
       v.test(npc.getHealth());
   }
   ```

## Data Pipeline
The DoubleSingleValidator functions as a gate within a larger data processing pipeline. It does not transform data; it either allows it to pass or signals an error, halting the pipeline for that particular asset.

> Flow:
> Raw Asset File (JSON/HOCON) -> Parser -> Raw Value (double) -> **DoubleSingleValidator** -> (If valid) -> NPC Property Assignment -> Finalized NPC Asset

