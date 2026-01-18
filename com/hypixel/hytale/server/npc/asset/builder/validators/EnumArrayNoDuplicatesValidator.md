---
description: Architectural reference for EnumArrayNoDuplicatesValidator
---

# EnumArrayNoDuplicatesValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class EnumArrayNoDuplicatesValidator extends EnumArrayValidator {
```

## Architecture & Concepts
The EnumArrayNoDuplicatesValidator is a specialized, stateless component within the server's asset validation framework. It inherits from EnumArrayValidator, signifying its role in a larger strategy pattern for validating different constraints on enum arrays.

Its sole responsibility is to enforce a uniqueness constraint. This validator is primarily invoked during the server's asset loading and building pipeline, particularly for Non-Player Character (NPC) definitions. When an NPC configuration is deserialized from a file, this validator ensures that fields defined as arrays of enums—such as a list of abilities, behaviors, or equipment slots—do not contain duplicate entries.

This preemptive validation is critical for maintaining data integrity and preventing subtle, difficult-to-diagnose runtime bugs that could arise from misconfigured game assets. It acts as a gatekeeper, ensuring that only valid asset configurations are loaded into the running game server.

### Lifecycle & Ownership
- **Creation:** The single instance is created eagerly and statically during class loading by the Java Virtual Machine. The private constructor and static `INSTANCE` field enforce the singleton pattern.
- **Scope:** The instance is application-scoped. It persists for the entire lifetime of the server process.
- **Destruction:** The object is managed by the JVM and is garbage collected during class unloading when the server shuts down. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields. Each invocation of the `test` method operates on a new, locally-scoped EnumSet, ensuring that calls do not interfere with one another.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless nature, multiple threads can safely call its methods concurrently without the need for external synchronization or locks. This makes it suitable for use in parallelized asset loading systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | EnumArrayNoDuplicatesValidator | O(1) | Returns the shared, static instance of the validator. |
| test(T[] array, Class<T> clazz) | boolean | O(N) | Validates that the input enum array contains no duplicate elements. Returns true if unique, false otherwise. N is the number of elements in the array. |
| errorMessage(String name, T[] array) | String | O(N) | Generates a formatted, human-readable error message for a failed validation. N is the number of elements in the array. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly. It is designed to be integrated into a higher-level asset builder or validation framework, where it is applied declaratively to a specific data field.

```java
// Hypothetical usage within an NPC Asset Builder
// The builder would retrieve the validator and apply it.

// Asset definition class
public class NpcDefinition {
    // This field must not contain duplicate abilities
    public Ability[] abilities;
}

// In the builder/validator logic
Validator<Ability[]> uniqueAbilitiesValidator = EnumArrayNoDuplicatesValidator.get();

boolean isValid = uniqueAbilitiesValidator.test(
    npcDefinition.abilities,
    Ability.class
);

if (!isValid) {
    throw new AssetValidationException(
        uniqueAbilitiesValidator.errorMessage("abilities", npcDefinition.abilities)
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Reflection-based Instantiation:** Do not attempt to bypass the private constructor using reflection. This violates the singleton pattern and provides no benefit, as the class is stateless. Always use the static `get` method.
- **Stateful Decoration:** Do not wrap this validator in another class that adds state. Its thread-safety and performance guarantees rely on its stateless design.

## Data Pipeline
The validator acts as a processing and validation step within a larger data transformation pipeline for game assets.

> Flow:
> NPC Asset File (JSON/HOCON) -> Deserializer -> Raw Data Object -> Asset Building Framework -> **EnumArrayNoDuplicatesValidator** -> Validation Result (Pass/Fail) -> Finalized, Engine-Ready NPC Asset

