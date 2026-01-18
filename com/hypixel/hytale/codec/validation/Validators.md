---
description: Architectural reference for Validators
---

# Validators

**Package:** com.hypixel.hytale.codec.validation
**Type:** Utility

## Definition
```java
// Signature
public class Validators {
```

## Architecture & Concepts
The Validators class is a static factory that serves as the central entry point for the engine's data validation framework. It provides a comprehensive suite of pre-built validation rules used to enforce data integrity, primarily within the codec (serialization/deserialization) system.

Its core architectural purpose is to decouple the definition of validation logic from its implementation. By providing a clean, declarative API, developers can specify constraints on data structures without needing to know the underlying validation mechanics. This approach promotes readable, maintainable, and robust data schemas.

The factory employs an optimization strategy by returning shared, stateless singleton instances for common validators (e.g., nonNull, nonEmptyString) while creating new, stateful instances for validators that require configuration (e.g., range, arraySize). This design minimizes object allocation for the most frequently used validation rules.

This class is a foundational component for ensuring that data loaded from disk or received from the network conforms to expected constraints before being hydrated into live game objects, preventing a wide range of bugs and potential security vulnerabilities.

### Lifecycle & Ownership
- **Creation:** The Validators class is never instantiated; it is a pure utility class containing only static methods. The individual Validator objects it produces are created on-demand when a corresponding factory method is invoked.
- **Scope:** As a static utility, the Validators class is available for the entire application lifetime. The Validator instances it returns are typically short-lived. They are created during the configuration of a codec or data schema and are held by that configuration object. Their lifecycle is tied to the schema they are part of.
- **Destruction:** Validator instances are managed by the Java garbage collector. They are eligible for cleanup once the codec or schema that references them is no longer in use.

## Internal State & Concurrency
- **State:** The Validators class is completely stateless. It maintains no internal fields or configuration. The Validator objects it returns are designed to be immutable after creation. For example, a RangeValidator created with `range(1, 100)` will hold the values 1 and 100 internally, but this state cannot be modified after instantiation.
- **Thread Safety:** This class is inherently thread-safe. All factory methods are re-entrant and free of side effects. The returned Validator objects are also thread-safe due to their immutable design, allowing a single codec configuration to be safely used across multiple threads for parallel data processing.

## API Surface
The API provides a set of static factory methods for creating Validator instances.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| nonNull() | Validator | O(1) | Returns a shared singleton validator that fails if the input object is null. |
| nonEmptyString() | Validator | O(1) | Returns a shared singleton validator that fails if a string is null or empty. |
| range(min, max) | Validator | O(1) | Creates a new validator that checks if a Comparable value is within the inclusive min/max bounds. |
| arraySize(size) | Validator | O(1) | Creates a new validator that checks if an array has an exact size. |
| requiredMapKeysValidator(keys) | Validator | O(1) | Creates a new validator that ensures a Map contains all specified keys. |
| or(validators) | Validator | O(N) | Creates a composite validator that passes if at least one of the provided inner validators passes. N is the number of validators. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use the static factory methods within the configuration of a data schema or codec. This allows for a declarative definition of data constraints.

```java
// Hypothetical usage during the construction of a data schema for a player profile.
// The validators ensure data integrity upon loading.

DataSchema playerProfileSchema = DataSchema.builder()
    .field("username", String.class, Validators.nonEmptyString())
    .field("level", Integer.class, Validators.range(1, 99))
    .field("skillRatings", int[].class, Validators.intArraySize(5))
    .field("lastKnownPosition", double[].class, Validators.doubleArraySize(3))
    .field("tags", String[].class, Validators.uniqueInArray())
    .build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate concrete validator implementations directly (e.g., `new NonNullValidator()`). The Validators factory is the sole intended entry point. Bypassing the factory breaks the abstraction and prevents the system from using optimized singleton instances.
- **Incorrect Validator Type:** Applying a validator to an incompatible data type will result in a ClassCastException at runtime. For example, using `nonEmptyString` on an Integer field is invalid.
- **Passing Null to Configuration:** Providing null arguments to stateful validators, such as `equal(null)`, may lead to a NullPointerException or undefined behavior. Refer to the Nonnull annotations for contract details.

## Data Pipeline
The Validators class does not process data directly. Instead, it manufactures the components (Validator instances) that are later used in a data processing pipeline, typically during deserialization.

> Flow:
> 1. **Configuration Time:** A developer uses the **Validators** factory to create and attach Validator instances to a schema or codec definition.
> 2. **Runtime:** A deserializer receives raw input (e.g., JSON, binary data).
> 3. **Execution:** As the deserializer processes fields, it invokes the `validate` method of the corresponding Validator instance attached to the schema.
> 4. **Decision:** If validation fails, the deserializer throws a validation exception, halting the object creation process. If it succeeds, processing continues.
> 5. **Output:** A fully validated and constructed game object.

