---
description: Architectural reference for NonEmptyFloatArrayValidator
---

# NonEmptyFloatArrayValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Singleton

## Definition
```java
// Signature
public class NonEmptyFloatArrayValidator implements Validator<float[]> {
```

## Architecture & Concepts
The NonEmptyFloatArrayValidator is a highly specialized, stateless component within the engine's data codec and validation framework. It embodies the Strategy Pattern, providing a concrete implementation for a single validation rule: ensuring a `float[]` is not null and contains at least one element.

This validator serves as a bridge between declarative data schemas and imperative code execution. During asset loading, configuration parsing, or network deserialization, a master validation service dispatches validation tasks to specific validators like this one based on schema annotations. Its primary role is to enforce data integrity at the boundaries of the system, preventing malformed or incomplete data from propagating into the game state.

## Lifecycle & Ownership
- **Creation:** The validator is a true singleton, instantiated by the Java ClassLoader when the NonEmptyFloatArrayValidator class is first referenced. The private constructor and public static final INSTANCE field enforce this pattern. There is no dynamic creation.
- **Scope:** Application-wide. The single instance persists for the entire lifetime of the application.
- **Destruction:** The instance is eligible for garbage collection only when the application's ClassLoader is unloaded, which typically occurs at shutdown.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is determined solely by its input arguments.
- **Thread Safety:** The component is inherently **thread-safe**. Its stateless nature allows it to be safely shared and executed by any number of threads concurrently without the need for locks or synchronization. This is critical for performance in parallelized asset loading pipelines.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(float[], ValidationResults) | void | O(1) | Executes the validation logic. Fails the provided ValidationResults if the input array is null or has a length of zero. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies a target ArraySchema to declaratively enforce the non-empty constraint by setting its minItems property to 1. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct invocation in most game logic. It is designed to be registered with and invoked by a higher-level validation service, often driven by a schema definition.

```java
// Hypothetical programmatic usage by a validation service
ValidationResults results = new ValidationResults();
float[] dataToValidate = new float[0]; // Invalid data

// The service would look up and apply the validator
NonEmptyFloatArrayValidator.INSTANCE.accept(dataToValidate, results);

if (results.hasFailed()) {
    // Handle validation failure
    System.out.println("Validation failed: " + results.getMessages());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to call `new NonEmptyFloatArrayValidator()` is a compile-time error and violates the singleton design. Always use the static `INSTANCE` field.
- **Ignoring Results:** The `accept` method does not throw exceptions; it mutates the passed-in ValidationResults object. Calling the method without subsequently checking the state of the results object renders the validation pointless.
- **Misapplication:** This validator is strongly typed for `float[]`. Do not attempt to use it for other data types; the system's generic type constraints should prevent this at compile time.

## Data Pipeline
This validator acts as a gate within a larger data deserialization and validation pipeline. It does not transform data but rather asserts conditions about it.

> Flow:
> Raw Data (e.g., JSON) -> Deserializer -> In-memory `float[]` -> Validation Service -> **NonEmptyFloatArrayValidator** -> Mutated ValidationResults -> Consumer Logic

